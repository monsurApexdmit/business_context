# Module Overview

**Module Name:** Shipping
**File Path:** `inventory_management/backend/shipping/shipping.md`
**Last Updated:** 2026-03-31

The Shipping module manages two related resources:

1. **Shipping Addresses** — saved delivery addresses associated with customers and companies.
2. **Order Shipments** — fulfillment records for orders, including carrier tracking details and a full tracking history log.

Additionally, the module exposes a public (unauthenticated) endpoint for tracking shipments by tracking number.

---

# Business Context

Shipping is a core fulfillment concern in the inventory management system. When an order (sell) is fulfilled, a shipment record is created and linked to that order. The shipment lifecycle progresses through a defined set of statuses, each of which drives updates to the linked order's fulfillment state. Customers or external parties can track their shipment publicly using a tracking number without needing to authenticate.

Shipping addresses provide reusable address records per customer, reducing data entry at checkout and enabling default address selection.

---

# Functional Scope

## Included

- CRUD operations for shipping addresses (per company, optionally per customer).
- Default address management per customer — only one address can be marked as default at a time.
- CRUD and status lifecycle management for order shipments.
- Automatic synchronization of the linked order (`sells`) fulfillment status on shipment creation and every status update.
- Shipment tracking history — an append-only log of tracking events per shipment.
- Manual tracking event insertion via a dedicated endpoint.
- Public shipment tracking lookup by tracking number.
- Aggregated shipment statistics per company.
- Pagination, filtering, and search on list endpoints.

## Excluded

- Carrier API integrations — no live carrier data pulls; tracking events are inserted manually or programmatically.
- Shipping rate calculation or cost estimation.
- Label generation.
- Address validation against external postal services.
- Multi-package shipments per order.

---

# Architecture Notes

- All authenticated endpoints are scoped to `company_id` extracted from the Bearer JWT. Cross-company data access is not possible.
- Shipment creation and the corresponding `sells` record update (tracking number, carrier, fulfillment status) occur within a single database transaction to ensure atomicity.
- Tracking history rows are ordered by `event_time DESC`.
- The `set-default` operation performs a two-step update: unset all existing defaults for the customer, then set the target address as default.
- The fulfillment status mapping from shipment status to sell's `fulfillment_status` is applied server-side on every status update. Clients cannot directly write the sell's fulfillment status through this module.
- The public tracking endpoint (`GET /track/:trackingNumber`) requires no authentication and returns only a safe subset of shipment data — internal fields such as `company_id`, `sell_id`, and cost information are not exposed.

---

# Data Model

## Table: `shipping_addresses`

| Column        | Type                          | Constraints              | JSON Key     |
|---------------|-------------------------------|--------------------------|--------------|
| id            | uint                          | PK, Auto Increment       | id           |
| company_id    | uint                          | FK → companies.id, Index | companyId    |
| customer_id   | *uint (nullable)              | FK → customers.id        | customerId   |
| full_name     | varchar(255)                  | Not null                 | fullName     |
| phone         | varchar(50)                   |                          | phone        |
| email         | varchar(255)                  |                          | email        |
| address_line1 | varchar(255)                  |                          | addressLine1 |
| address_line2 | varchar(255)                  |                          | addressLine2 |
| city          | varchar(100)                  |                          | city         |
| state         | varchar(100)                  |                          | state        |
| postal_code   | varchar(20)                   |                          | postalCode   |
| country       | varchar(100)                  | Default: `'Bangladesh'`  | country      |
| is_default    | bool                          | Default: `false`         | isDefault    |
| address_type  | enum('home','office','other') |                          | addressType  |
| created_at    | timestamp                     | Auto-managed             | createdAt    |
| updated_at    | timestamp                     | Auto-managed             | updatedAt    |
| deleted_at    | timestamp (nullable)          | Soft delete              | —            |

## Table: `order_shipments`

| Column             | Type                 | Constraints                     | JSON Key          |
|--------------------|----------------------|---------------------------------|-------------------|
| id                 | uint                 | PK, Auto Increment              | id                |
| company_id         | uint                 | FK → companies.id               | companyId         |
| sell_id            | uint                 | FK → sells.id                   | sellId            |
| tracking_number    | varchar(100)         |                                 | trackingNumber    |
| carrier            | varchar(100)         |                                 | carrier           |
| shipping_method    | varchar(100)         |                                 | shippingMethod    |
| status             | varchar(50)          | See valid statuses below        | status            |
| shipped_at         | *timestamp (nullable)|                                 | shippedAt         |
| estimated_delivery | *timestamp (nullable)|                                 | estimatedDelivery |
| delivered_at       | *timestamp (nullable)|                                 | deliveredAt       |
| shipping_cost      | float64              | Default: `0`                    | shippingCost      |
| weight             | float64              |                                 | weight            |
| dimensions         | string               |                                 | dimensions        |
| notes              | text                 |                                 | notes             |
| created_at         | timestamp            | Auto-managed                    | createdAt         |
| updated_at         | timestamp            | Auto-managed                    | updatedAt         |

**Valid shipment statuses:** `pending`, `picked_up`, `in_transit`, `out_for_delivery`, `delivered`, `failed`, `returned`

## Table: `shipment_tracking_history`

| Column      | Type      | Constraints              | JSON Key    |
|-------------|-----------|--------------------------|-------------|
| id          | uint      | PK, Auto Increment       | id          |
| shipment_id | uint      | FK → order_shipments.id  | shipmentId  |
| status      | string    |                          | status      |
| location    | string    |                          | location    |
| description | string    |                          | description |
| event_time  | timestamp |                          | eventTime   |
| created_at  | timestamp | Auto-managed             | createdAt   |

## Relationships

| From              | Relationship | To               | Notes                                    |
|-------------------|--------------|------------------|------------------------------------------|
| ShippingAddress   | BelongsTo    | Customer         | Optional — `customer_id` is nullable     |
| OrderShipment     | BelongsTo    | Sell             | Required                                 |
| OrderShipment     | HasMany      | TrackingHistory  | Ordered by `event_time DESC`             |

---

# API Endpoints

## Shipping Addresses

| Method | URL                                   | Auth       | Purpose                                  |
|--------|---------------------------------------|------------|------------------------------------------|
| GET    | `/shipping-addresses/`                | Bearer JWT | List shipping addresses with filters     |
| GET    | `/shipping-addresses/:id`             | Bearer JWT | Get a single shipping address by ID      |
| POST   | `/shipping-addresses/`                | Bearer JWT | Create a new shipping address            |
| PUT    | `/shipping-addresses/:id`             | Bearer JWT | Update an existing shipping address      |
| DELETE | `/shipping-addresses/:id`             | Bearer JWT | Soft-delete a shipping address           |
| PATCH  | `/shipping-addresses/:id/set-default` | Bearer JWT | Set an address as the customer's default |

## Order Shipments

| Method | URL                         | Auth           | Purpose                                    |
|--------|-----------------------------|----------------|--------------------------------------------|
| GET    | `/shipments/`               | Bearer JWT     | List shipments with pagination and filters |
| GET    | `/shipments/:id`            | Bearer JWT     | Get a single shipment with full details    |
| POST   | `/shipments/`               | Bearer JWT     | Create a new shipment                      |
| PATCH  | `/shipments/:id/status`     | Bearer JWT     | Update shipment status                     |
| POST   | `/shipments/:id/tracking`   | Bearer JWT     | Manually add a tracking history event      |
| GET    | `/shipments/stats`          | Bearer JWT     | Get aggregated shipment statistics         |
| GET    | `/track/:trackingNumber`    | None (public)  | Public shipment tracking lookup            |

---

# Request Payloads

## `GET /shipping-addresses/` — Query Parameters

| Parameter    | Type   | Required | Description                               |
|--------------|--------|----------|-------------------------------------------|
| customer_id  | uint   | No       | Filter by customer ID                     |
| address_type | string | No       | Filter by type: `home`, `office`, `other` |
| is_default   | string | No       | Pass `"true"` to return only defaults     |

---

## `POST /shipping-addresses/` — Request Body

**Content-Type:** `application/json`

| Field        | Type   | Required | Description                              |
|--------------|--------|----------|------------------------------------------|
| customerId   | uint   | No       | Associated customer ID                   |
| fullName     | string | Yes      | Full name of recipient                   |
| phone        | string | Yes      | Contact phone number                     |
| email        | string | No       | Contact email address                    |
| addressLine1 | string | Yes      | Primary address line                     |
| addressLine2 | string | No       | Secondary address line (apartment, etc.) |
| city         | string | Yes      | City                                     |
| state        | string | Yes      | State or division                        |
| postalCode   | string | Yes      | Postal or ZIP code                       |
| country      | string | No       | Country name (default: `"Bangladesh"`)   |
| isDefault    | bool   | No       | Whether this is the default address      |
| addressType  | string | No       | `home`, `office`, or `other`             |

---

## `PUT /shipping-addresses/:id` — Request Body

**Content-Type:** `application/json`

Any subset of the fields listed for `POST /shipping-addresses/`. Partial updates are supported.

---

## `GET /shipments/` — Query Parameters

| Parameter       | Type   | Required | Default | Description                                                                                                                               |
|-----------------|--------|----------|---------|-------------------------------------------------------------------------------------------------------------------------------------------|
| page            | int    | No       | 1       | Page number                                                                                                                               |
| per_page        | int    | No       | 10      | Records per page                                                                                                                          |
| sell_id         | uint   | No       | —       | Filter by sell/order ID                                                                                                                   |
| status          | string | No       | —       | Filter by status. Valid values: `pending`, `picked_up`, `in_transit`, `out_for_delivery`, `delivered`, `failed`, `returned`. Use `"all"` to skip. |
| tracking_number | string | No       | —       | Partial match (LIKE) on tracking number                                                                                                   |
| carrier         | string | No       | —       | Exact match on carrier name                                                                                                               |

---

## `POST /shipments/` — Request Body

**Content-Type:** `application/json`

| Field             | Type      | Required | Description                                         |
|-------------------|-----------|----------|-----------------------------------------------------|
| sellId            | uint      | Yes      | ID of the associated order (sell)                   |
| trackingNumber    | string    | Yes      | Carrier tracking number                             |
| carrier           | string    | Yes      | Carrier name (e.g., `"DHL"`, `"FedEx"`)             |
| shippingMethod    | string    | No       | Shipping method description (e.g., `"Express"`)     |
| status            | string    | No       | Initial status — must be valid; defaults to `pending` |
| shippedAt         | timestamp | No       | Timestamp when the shipment was handed to carrier   |
| estimatedDelivery | timestamp | No       | Estimated delivery timestamp                        |
| shippingCost      | float64   | No       | Cost of shipping                                    |
| weight            | float64   | No       | Package weight                                      |
| dimensions        | string    | No       | Package dimensions as free-form string              |
| notes             | string    | No       | Internal notes                                      |

---

## `PATCH /shipments/:id/status` — Request Body

**Content-Type:** `application/json`

| Field       | Type   | Required | Description                                                 |
|-------------|--------|----------|-------------------------------------------------------------|
| status      | string | Yes      | New shipment status — must be a valid status value          |
| location    | string | No       | Current physical location of the shipment                   |
| description | string | No       | Event description — auto-generated by server if not provided |

---

## `POST /shipments/:id/tracking` — Request Body

**Content-Type:** `application/json`

| Field       | Type      | Required | Description                            |
|-------------|-----------|----------|----------------------------------------|
| status      | string    | No       | Status label for this tracking event   |
| location    | string    | No       | Location at the time of the event      |
| description | string    | No       | Description of the tracking event      |
| eventTime   | timestamp | No       | Timestamp of the event                 |

---

# Response Contracts

## `GET /shipping-addresses/` — 200 OK

```json
{
  "message": "Shipping addresses retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "customerId": 5,
      "customer": { "id": 5, "name": "..." },
      "fullName": "John Doe",
      "phone": "+1234567890",
      "email": "john@example.com",
      "addressLine1": "123 Main St",
      "addressLine2": "",
      "city": "Dhaka",
      "state": "Dhaka Division",
      "postalCode": "1200",
      "country": "Bangladesh",
      "isDefault": true,
      "addressType": "home",
      "createdAt": "2026-01-01T00:00:00Z",
      "updatedAt": "2026-01-01T00:00:00Z"
    }
  ]
}
```

## `GET /shipping-addresses/:id` — 200 OK

```json
{
  "message": "Shipping address fetched successfully",
  "data": { /* shipping address object with customer preloaded */ }
}
```

## `POST /shipping-addresses/` — 201 Created

```json
{
  "message": "Shipping address created successfully",
  "data": { /* shipping address object with customer preloaded */ }
}
```

## `PUT /shipping-addresses/:id` — 200 OK

```json
{
  "message": "Shipping address updated successfully",
  "data": { /* shipping address object with customer preloaded */ }
}
```

## `DELETE /shipping-addresses/:id` — 200 OK

```json
{
  "message": "Shipping address deleted successfully"
}
```

## `PATCH /shipping-addresses/:id/set-default` — 200 OK

```json
{
  "message": "Default shipping address updated successfully",
  "data": { /* shipping address object */ }
}
```

## `GET /shipments/` — 200 OK

```json
{
  "message": "Shipments retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "sellId": 3,
      "sell": { "id": 3, "..." : "..." },
      "trackingNumber": "TRACK123",
      "carrier": "FedEx",
      "shippingMethod": "Express",
      "status": "in_transit",
      "shippedAt": "2026-03-01T10:00:00Z",
      "estimatedDelivery": "2026-03-05T00:00:00Z",
      "deliveredAt": null,
      "shippingCost": 12.99,
      "weight": 1.5,
      "dimensions": "30x20x10",
      "notes": "",
      "trackingHistory": [ { "..." : "..." } ],
      "createdAt": "2026-03-01T09:00:00Z",
      "updatedAt": "2026-03-02T08:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 10,
    "total": 25
  }
}
```

## `GET /shipments/:id` — 200 OK

```json
{
  "message": "Shipment fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "sellId": 3,
    "sell": { "..." : "..." },
    "trackingNumber": "TRACK123",
    "carrier": "FedEx",
    "shippingMethod": "Express",
    "status": "in_transit",
    "shippedAt": "2026-03-01T10:00:00Z",
    "estimatedDelivery": "2026-03-05T00:00:00Z",
    "deliveredAt": null,
    "shippingCost": 12.99,
    "weight": 1.5,
    "dimensions": "30x20x10",
    "notes": "",
    "trackingHistory": [
      {
        "id": 2,
        "shipmentId": 1,
        "status": "in_transit",
        "location": "Chittagong Hub",
        "description": "Package in transit",
        "eventTime": "2026-03-02T08:00:00Z",
        "createdAt": "2026-03-02T08:05:00Z"
      }
    ],
    "createdAt": "2026-03-01T09:00:00Z",
    "updatedAt": "2026-03-02T08:00:00Z"
  }
}
```

## `POST /shipments/` — 201 Created

```json
{
  "message": "Shipment created successfully",
  "data": { /* shipment object with sell and trackingHistory preloaded */ }
}
```

## `PATCH /shipments/:id/status` — 200 OK

```json
{
  "message": "Shipment status updated successfully",
  "data": { /* shipment object with updated status and tracking history */ }
}
```

## `POST /shipments/:id/tracking` — 201 Created

```json
{
  "message": "Tracking event added successfully",
  "data": {
    "id": 5,
    "shipmentId": 1,
    "status": "in_transit",
    "location": "Chittagong Hub",
    "description": "Package arrived at hub",
    "eventTime": "2026-03-02T08:00:00Z",
    "createdAt": "2026-03-02T08:05:00Z"
  }
}
```

## `GET /shipments/stats` — 200 OK

```json
{
  "message": "Shipment statistics retrieved successfully",
  "data": {
    "total": 50,
    "pending": 10,
    "inTransit": 15,
    "delivered": 20,
    "failed": 5,
    "totalShippingCost": 649.50,
    "avgDeliveryDays": 3.2
  }
}
```

## `GET /track/:trackingNumber` — 200 OK (Public)

```json
{
  "message": "Shipment tracking retrieved successfully",
  "data": {
    "trackingNumber": "TRACK123",
    "carrier": "FedEx",
    "status": "delivered",
    "shippedAt": "2026-03-01T10:00:00Z",
    "estimatedDelivery": "2026-03-05T00:00:00Z",
    "deliveredAt": "2026-03-04T14:00:00Z",
    "trackingHistory": [
      {
        "status": "delivered",
        "location": "Dhaka",
        "description": "Package delivered",
        "eventTime": "2026-03-04T14:00:00Z"
      }
    ]
  }
}
```

---

# Validation Rules

## Shipping Addresses

| Field        | Rule                                                                             |
|--------------|----------------------------------------------------------------------------------|
| fullName     | Required, non-empty string                                                       |
| phone        | Required, non-empty string                                                       |
| addressLine1 | Required, non-empty string                                                       |
| city         | Required, non-empty string                                                       |
| state        | Required, non-empty string                                                       |
| postalCode   | Required, non-empty string                                                       |
| addressType  | If provided, must be one of: `home`, `office`, `other`                           |
| set-default  | Address must already have a `customer_id` set; returns `400` if missing          |

## Order Shipments

| Field          | Rule                                                                                     |
|----------------|------------------------------------------------------------------------------------------|
| sellId         | Required on create; referenced sell must exist within the company                        |
| trackingNumber | Required on create                                                                       |
| carrier        | Required on create                                                                       |
| status         | If provided on create, must be one of the valid status values; defaults to `pending`     |
| status (PATCH) | Required; returns `400` if missing or not a recognized status value                      |

---

# Business Logic Details

## Shipping Addresses — Default Management

- **On create:** If `isDefault = true` and `customerId` is provided, all other addresses for that customer under the same company are set to `isDefault = false` before the new record is saved.
- **On update:** Same default-toggling logic applies if `isDefault = true` is included in the update payload.
- **`PATCH /shipping-addresses/:id/set-default`:** Explicitly designates one address as the customer's default. All other addresses for the same customer are unset. Returns `400` if the target address has no `customer_id`.

## Order Shipments — Creation

1. Validates the referenced `sell` exists for the company. Returns `404` if not found.
2. Creates the `order_shipments` record.
3. Within the same database transaction:
   - Updates the linked `sells` record: sets `tracking_number`, `carrier`, and `fulfillment_status`.
   - Inserts an initial tracking history event.
4. Returns the created shipment with `sell` and `trackingHistory` preloaded.

## Order Shipments — Status Update (`PATCH /shipments/:id/status`)

Timestamp side effects triggered by status transition:

| New Status    | Timestamp Behavior                                 |
|---------------|----------------------------------------------------|
| `picked_up`   | Sets `shipped_at` if not already set               |
| `delivered`   | Sets `delivered_at` if not already set             |
| All others    | No timestamp changes                               |

Fulfillment status sync applied to the linked `sells` record on every status update:

| Shipment Status                              | Sell `fulfillment_status` |
|----------------------------------------------|---------------------------|
| `pending`                                    | `processing`              |
| `picked_up`, `in_transit`, `out_for_delivery`| `shipped`                 |
| `delivered`                                  | `delivered`               |
| `failed`, `returned`                         | `cancelled`               |
| Unrecognized / default                       | `unfulfilled`             |

A new tracking history event is always appended on each status update. If `description` is not provided in the request body, the server auto-generates one.

---

# Dependencies

| Module      | Direction | Notes                                                                                         |
|-------------|-----------|-----------------------------------------------------------------------------------------------|
| `customers` | Outbound  | `shipping_addresses.customer_id` references `customers.id`; customer preloaded on reads       |
| `sells`     | Outbound  | `order_shipments.sell_id` references `sells.id`; sell preloaded on reads and updated on shipment creation and status changes |
| `companies` | Outbound  | All records are scoped by `company_id` extracted from the JWT                                 |

---

# Security / Permissions

- All endpoints except `GET /track/:trackingNumber` require a valid Bearer JWT.
- `company_id` is extracted from the JWT and enforced on every database query. Clients cannot access or modify data belonging to other companies.
- The public tracking endpoint (`GET /track/:trackingNumber`) returns only a safe subset of shipment data: `trackingNumber`, `carrier`, `status`, `shippedAt`, `estimatedDelivery`, `deliveredAt`, and `trackingHistory`. Internal fields (e.g., `company_id`, `sell_id`, `shippingCost`) are not exposed.
- No role-level access control (RBAC) is documented for this module beyond JWT authentication.

---

# Error Handling

## Shipping Addresses

| HTTP Status | Trigger                                                               |
|-------------|-----------------------------------------------------------------------|
| 400         | Missing required fields on create (`fullName`, `phone`, `addressLine1`, `city`, `state`, `postalCode`) |
| 400         | Invalid request body (malformed JSON)                                 |
| 400         | `set-default` called on an address that has no `customer_id`          |
| 404         | Shipping address not found for the given `id` within the company      |
| 500         | Unexpected server or database error                                   |

**Error response shape:**
```json
{ "error": "Human-readable error message" }
```

## Order Shipments

| HTTP Status | Trigger                                                                           |
|-------------|-----------------------------------------------------------------------------------|
| 400         | Missing required fields on create (`sellId`, `trackingNumber`, `carrier`)         |
| 400         | Invalid or unrecognized `status` value                                            |
| 400         | `status` field missing on `PATCH /shipments/:id/status`                           |
| 400         | Invalid request body (malformed JSON)                                             |
| 404         | Order (sell) not found on shipment create                                         |
| 404         | Shipment not found for the given `id`                                             |
| 404         | Tracking number not found on public tracking endpoint                             |
| 500         | Unexpected server or database error                                               |

---

# Testing Notes

- **Transaction atomicity:** Verify that creating a shipment updates the linked `sell`'s `tracking_number`, `carrier`, and `fulfillment_status` as an atomic transaction. A database failure mid-way should roll back both the shipment record and the sell update.
- **Default address toggle:** Creating or updating an address with `isDefault = true` must unset all other addresses for the same customer. Verify no two addresses for the same customer share `isDefault = true`.
- **`set-default` without customer:** Confirm `PATCH /shipping-addresses/:id/set-default` returns `400` when the address has no `customer_id`.
- **Status transitions:** Test all seven status values on `PATCH /shipments/:id/status` and verify the correct `fulfillment_status` is written to the sell.
- **Timestamp idempotency:** Verify `shipped_at` is set only once (on the first `picked_up` transition) and `delivered_at` is set only once (on the first `delivered` transition). Subsequent transitions to the same status should not overwrite them.
- **Auto-generated description:** Confirm that omitting `description` on a status update results in a non-empty auto-generated description in the tracking history event.
- **Public tracking endpoint:** Test `GET /track/:trackingNumber` without any Authorization header — must return `200` and must not expose `company_id`, `sell_id`, or `shippingCost`.
- **Pagination and filtering:** Test `page`, `per_page`, `status`, `tracking_number`, and `carrier` filters on `GET /shipments/`.
- **Company scoping:** Confirm that authenticated requests return only records belonging to the company in the JWT. Attempts to access another company's records must return `404`.
- **Stats aggregation:** Verify `GET /shipments/stats` returns correct aggregate counts and cost totals for the company.

---

# Open Questions / Missing Details

- **Auto-generated description format:** When `description` is omitted on a status update, the server auto-generates one. The template or format of this generated string is not documented.
- **`sells.fulfillment_status` column:** The exact column name and the full set of valid values for `fulfillment_status` on the `sells` table are inferred from business logic but not explicitly defined in this module's spec.
- **Pagination on `GET /shipping-addresses/`:** No pagination parameters are documented for the address list endpoint. It is unclear whether the endpoint returns all matching records or supports paging.
- **`POST /shipments/:id/tracking` — `eventTime` default:** If `eventTime` is not provided, it is unclear whether it defaults to the current server timestamp or is stored as null.
- **Soft delete on `order_shipments`:** The schema does not mention soft delete for this table. Confirm whether deletion (if supported) uses hard or soft delete.
- **RBAC beyond JWT:** It is unknown whether specific staff roles or permission flags restrict access to any shipping endpoints beyond requiring a valid JWT.
- **Shipment stats filter support:** It is unknown whether `GET /shipments/stats` supports query parameters such as date range filters.
- **Shipment update endpoint:** A full `PUT /shipments/:id` endpoint is not documented. Confirm whether updating non-status fields (e.g., `shippingCost`, `weight`, `notes`) after creation is supported.

---

# Change Log / Notes

| Date       | Author | Note                                                            |
|------------|--------|-----------------------------------------------------------------|
| 2026-03-31 | System | Initial standardized documentation created from source spec     |
