# Module: Shipping

## Business Purpose
Manages two related resources: (1) **Shipping Addresses** — saved delivery addresses for customers, and (2) **Order Shipments** — fulfillment records for orders with carrier tracking and history. Also provides a public tracking endpoint for shipment lookup by tracking number.

---

## Database Tables

### Table: `shipping_addresses`
| Column          | Type          | Notes                                                         |
|-----------------|---------------|---------------------------------------------------------------|
| id              | uint (PK, AI) |                                                               |
| company_id      | uint (FK)     | FK → companies.id; indexed                                    |
| customer_id     | *uint (FK)    | FK → customers.id; nullable                                   |
| full_name       | varchar(255)  | json: fullName                                                |
| phone           | varchar(50)   | json: phone                                                   |
| email           | varchar(255)  | json: email                                                   |
| address_line1   | varchar(255)  | json: addressLine1                                            |
| address_line2   | varchar(255)  | json: addressLine2                                            |
| city            | varchar(100)  | json: city                                                    |
| state           | varchar(100)  | json: state                                                   |
| postal_code     | varchar(20)   | json: postalCode                                              |
| country         | varchar(100)  | default 'Bangladesh'; json: country                           |
| is_default      | bool          | default false; json: isDefault                                |
| address_type    | enum          | 'home', 'office', 'other'; json: addressType                  |
| created_at      | timestamp     | json: createdAt                                               |
| updated_at      | timestamp     | json: updatedAt                                               |
| deleted_at      | timestamp     | Soft delete (json: "-")                                       |

### Table: `order_shipments`
| Column              | Type          | Notes                                                                    |
|---------------------|---------------|--------------------------------------------------------------------------|
| id                  | uint (PK, AI) |                                                                          |
| company_id          | uint (FK)     | FK → companies.id                                                        |
| sell_id             | uint (FK)     | FK → sells.id; json: sellId                                              |
| tracking_number     | varchar(100)  | json: trackingNumber                                                     |
| carrier             | varchar(100)  | json: carrier                                                            |
| shipping_method     | varchar(100)  | json: shippingMethod                                                     |
| status              | varchar(50)   | 'pending','picked_up','in_transit','out_for_delivery','delivered','failed','returned'; json: status |
| shipped_at          | *timestamp    | nullable; json: shippedAt                                                |
| estimated_delivery  | *timestamp    | nullable; json: estimatedDelivery                                        |
| delivered_at        | *timestamp    | nullable; json: deliveredAt                                              |
| shipping_cost       | float64       | default 0; json: shippingCost                                            |
| weight              | float64       | json: weight                                                             |
| dimensions          | string        | json: dimensions                                                         |
| notes               | text          | json: notes                                                              |
| created_at          | timestamp     | json: createdAt                                                          |
| updated_at          | timestamp     | json: updatedAt                                                          |

### Table: `shipment_tracking_history`
| Column      | Type          | Notes                             |
|-------------|---------------|-----------------------------------|
| id          | uint (PK, AI) |                                   |
| shipment_id | uint (FK)     | FK → order_shipments.id           |
| status      | string        | json: status                      |
| location    | string        | json: location                    |
| description | string        | json: description                 |
| event_time  | timestamp     | json: eventTime                   |
| created_at  | timestamp     | json: createdAt                   |

---

## Relationships
- ShippingAddress → Customer (BelongsTo, optional)
- OrderShipment → Sell (BelongsTo)
- OrderShipment → TrackingHistory (HasMany, ordered by event_time DESC)

---

## Business Logic

### Shipping Addresses
- On **create**: if `is_default = true` and `customer_id` is set, all other addresses for that customer are set to `is_default = false` first.
- On **update**: same default-toggling behavior.
- `SetDefaultShippingAddress`: explicitly sets one address as default and unsets others for the same customer.

### Order Shipments
- On **create**: shipment record is created; sell's `tracking_number`, `carrier`, `fulfillment_status` are updated in the same transaction; an initial tracking history event is added.
- On **UpdateShipmentStatus**:
  - `picked_up` → sets `shipped_at` if not already set
  - `delivered` → sets `delivered_at` if not already set
  - Sell's `fulfillment_status` is updated to match (see mapping below)
  - A new tracking history event is added
- Fulfillment status mapping from shipment status:
  - `pending` → `"processing"`
  - `picked_up`, `in_transit`, `out_for_delivery` → `"shipped"`
  - `delivered` → `"delivered"`
  - `failed`, `returned` → `"cancelled"`
  - default → `"unfulfilled"`

---

## Endpoints — Shipping Addresses

### GET /shipping-addresses/
**Auth:** Bearer JWT required

**Query Parameters:**
| Param        | Type   | Description                          |
|--------------|--------|--------------------------------------|
| customer_id  | uint   | Filter by customer                   |
| address_type | string | Filter by type (home/office/other)   |
| is_default   | string | "true" to return only default addresses |

**Response 200:**
```json
{
  "message": "Shipping addresses retrieved successfully",
  "data": [
    {
      "id": 1,
      "companyId": 1,
      "customerId": 5,
      "customer": { ... },
      "fullName": "John Doe",
      "phone": "+1234567890",
      "email": "john@example.com",
      "addressLine1": "123 Main St",
      "addressLine2": "",
      "city": "New York",
      "state": "NY",
      "postalCode": "10001",
      "country": "US",
      "isDefault": true,
      "addressType": "home",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

---

### GET /shipping-addresses/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Shipping address fetched successfully",
  "data": { /* shipping address with customer preloaded */ }
}
```

**Error:** `404` — `{"error": "Shipping address not found"}`

---

### POST /shipping-addresses/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "customerId": 5,
  "fullName": "John Doe",
  "phone": "+1234567890",
  "email": "john@example.com",
  "addressLine1": "123 Main St",
  "addressLine2": "",
  "city": "New York",
  "state": "NY",
  "postalCode": "10001",
  "country": "US",
  "isDefault": true,
  "addressType": "home"
}
```

**Validation:**
- `fullName` — required
- `phone` — required
- `addressLine1` — required
- `city` — required
- `state` — required
- `postalCode` — required

**Response 201:**
```json
{
  "message": "Shipping address created successfully",
  "data": { /* shipping address with customer preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Full name is required"}` / `{"error": "Phone is required"}` / etc.
- `500` — `{"error": "Failed to create shipping address"}`

---

### PUT /shipping-addresses/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** Any subset of shipping address fields.

**Response 200:**
```json
{
  "message": "Shipping address updated successfully",
  "data": { /* shipping address with customer preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `404` — `{"error": "Shipping address not found"}`
- `500` — `{"error": "Failed to update shipping address"}`

---

### DELETE /shipping-addresses/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Shipping address deleted successfully"
}
```

**Error:** `404` — `{"error": "Shipping address not found"}`

---

### PATCH /shipping-addresses/:id/set-default
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Default shipping address updated successfully",
  "data": { /* shipping address object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Cannot set default for address without customer"}`
- `404` — `{"error": "Shipping address not found"}`

---

## Endpoints — Order Shipments

### GET /shipments/
**Auth:** Bearer JWT required

**Query Parameters:**
| Param           | Type   | Default | Description                                              |
|-----------------|--------|---------|----------------------------------------------------------|
| page            | int    | 1       |                                                          |
| per_page        | int    | 10      |                                                          |
| sell_id         | uint   |         | Filter by order                                          |
| status          | string |         | pending/picked_up/in_transit/out_for_delivery/delivered/failed/returned; "all" = no filter |
| tracking_number | string |         | LIKE search on tracking number                           |
| carrier         | string |         | Filter by carrier                                        |

**Response 200:**
```json
{
  "message": "Shipments retrieved successfully",
  "data": [ /* array of shipment objects with sell and tracking history */ ],
  "pagination": { "page": 1, "per_page": 10, "total": 25 }
}
```

---

### GET /shipments/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Shipment fetched successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "sellId": 3,
    "sell": { ... },
    "trackingNumber": "TRACK123",
    "carrier": "FedEx",
    "shippingMethod": "Express",
    "status": "in_transit",
    "shippedAt": "2024-01-02T10:00:00Z",
    "estimatedDelivery": "2024-01-05T00:00:00Z",
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
        "location": "Chicago Hub",
        "description": "Package in transit",
        "eventTime": "2024-01-03T08:00:00Z"
      }
    ],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**Error:** `404` — `{"error": "Shipment not found"}`

---

### POST /shipments/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "sellId": 3,
  "trackingNumber": "TRACK123",
  "carrier": "FedEx",
  "shippingMethod": "Express",
  "status": "pending",
  "shippedAt": "2024-01-02T10:00:00Z",
  "estimatedDelivery": "2024-01-05T00:00:00Z",
  "shippingCost": 12.99,
  "weight": 1.5,
  "dimensions": "30x20x10",
  "notes": ""
}
```

**Validation:**
- `sellId` — required (must reference an existing order)
- `trackingNumber` — required
- `carrier` — required
- `status` — optional; must be one of valid statuses; defaults to "pending"

**Response 201:**
```json
{
  "message": "Shipment created successfully",
  "data": { /* shipment with sell and tracking history */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `400` — `{"error": "Order ID (sellId) is required"}` / `{"error": "Tracking number is required"}` / `{"error": "Carrier is required"}`
- `400` — `{"error": "Invalid shipment status"}`
- `404` — `{"error": "Order not found"}`
- `500` — `{"error": "Failed to create shipment"}`

---

### PATCH /shipments/:id/status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "status": "delivered",
  "location": "New York",
  "description": "Package delivered to front door"
}
```

**Validation:**
- `status` — required; must be one of valid statuses
- `location` — optional
- `description` — optional (auto-generates if empty)

**Response 200:**
```json
{
  "message": "Shipment status updated successfully",
  "data": { /* shipment with tracking history */ }
}
```

**Error Responses:**
- `400` — `{"error": "Status is required"}` / `{"error": "Invalid status"}`
- `404` — `{"error": "Shipment not found"}`
- `500` — `{"error": "Failed to update shipment status"}`

---

### POST /shipments/:id/tracking
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "status": "in_transit",
  "location": "Chicago Hub",
  "description": "Package arrived at sorting facility",
  "eventTime": "2024-01-03T08:00:00Z"
}
```

**Response 201:**
```json
{
  "message": "Tracking event added successfully",
  "data": { /* ShipmentTrackingHistory object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body"}`
- `404` — `{"error": "Shipment not found"}`
- `500` — `{"error": "Failed to add tracking event"}`

---

### GET /track/:trackingNumber
**Auth:** None (public endpoint)
**Purpose:** Public shipment tracking by tracking number.

**Response 200:**
```json
{
  "message": "Shipment tracking retrieved successfully",
  "data": {
    "trackingNumber": "TRACK123",
    "carrier": "FedEx",
    "status": "delivered",
    "shippedAt": "2024-01-02T10:00:00Z",
    "estimatedDelivery": "2024-01-05T00:00:00Z",
    "deliveredAt": "2024-01-04T14:00:00Z",
    "trackingHistory": [ ... ]
  }
}
```

**Error:** `404` — `{"error": "Shipment not found"}`

---

### GET /shipments/stats
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Shipment statistics retrieved successfully",
  "data": {
    "total": 50,
    "pending": 10,
    "inTransit": 15,
    "delivered": 20,
    "failed": 5,
    "totalShippingCost": 649.5,
    "avgDeliveryDays": 3.2
  }
}
```

---

## Dependencies
- **customers** — shipping addresses reference customers
- **orders** — shipments reference orders (sells)
