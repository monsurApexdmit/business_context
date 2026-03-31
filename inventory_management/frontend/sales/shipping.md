# Frontend Module: Shipping / Shipments

## Pages
- `/dashboard/sales/shipments` — Shipment list with status filters and stats
- `/dashboard/sales/shipments/new` — Create shipment for an order
- `/dashboard/sales/shipments/:id` — Shipment detail with tracking history

## API Service
`lib/shipmentsApi.ts`

---

## GET /shipments
**Purpose:** Fetch paginated list of shipments with optional filters
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)
- `page` — page number (optional)
- `limit` — items per page (optional)
- `search` — search by tracking number, carrier, or order invoice (optional)
- `status` — filter by shipment status (optional): `"pending"` | `"picked_up"` | `"in_transit"` | `"out_for_delivery"` | `"delivered"` | `"failed"` | `"returned"`
- `carrier` — filter by carrier name (optional)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": [
    {
      "id": 1,
      "sellId": 10,
      "sell": {
        "id": 10,
        "invoiceNo": "INV-20240101-001",
        "customerName": "John Doe",
        "method": "card",
        "amount": 99.99,
        "status": "Processing",
        "shippedAt": "2024-01-02T00:00:00Z"
      },
      "trackingNumber": "1Z999AA1012345678",
      "carrier": "UPS",
      "shippingMethod": "ground",
      "status": "in_transit",
      "estimatedDelivery": "2024-01-05T00:00:00Z",
      "shippingCost": 8.50,
      "weight": 1.5,
      "dimensions": "10x8x4",
      "notes": "string",
      "shippedAt": "2024-01-02T00:00:00Z",
      "deliveredAt": null,
      "trackingHistory": [
        {
          "id": 1,
          "shipmentId": 1,
          "status": "picked_up",
          "location": "Chicago, IL",
          "description": "Package picked up",
          "eventTime": "2024-01-02T09:00:00Z",
          "createdAt": "2024-01-02T09:00:00Z"
        }
      ],
      "createdAt": "2024-01-01T00:00:00Z",
      "updatedAt": "2024-01-02T09:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 75
  }
}
```

**Note:** Pagination uses `per_page` (not `limit`) in the response object.

**Frontend Impact:**
- Shipment list with status filter tabs
- Tracking number displayed as copyable field
- `sell.invoiceNo` links back to the related order

---

## GET /shipments/stats
**Purpose:** Fetch aggregate shipment counts by status
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "total": 75,
    "pending": 10,
    "in_transit": 25,
    "delivered": 35,
    "failed": 3,
    "picked_up": 0,
    "out_for_delivery": 2,
    "returned": 0
  }
}
```

**Frontend Impact:**
- Stats cards at top of shipments page
- Filter tab counts (Pending: 10, In Transit: 25, Delivered: 35)

---

## GET /shipments/:id
**Purpose:** Fetch full shipment details including complete tracking history
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric shipment ID

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...full ShipmentResponse object with trackingHistory array..." }
}
```

**Frontend Impact:**
- Shipment detail page with full tracking timeline
- `trackingHistory` rendered as a vertical timeline component
- Estimated delivery date shown prominently

---

## POST /shipments
**Purpose:** Create a new shipment for an existing order
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "sellId": 10,
  "trackingNumber": "1Z999AA1012345678",
  "carrier": "UPS",
  "shippingMethod": "ground",
  "status": "pending",
  "estimatedDelivery": "2024-01-05T00:00:00Z",
  "shippingCost": 8.50,
  "weight": 1.5,
  "dimensions": "10x8x4",
  "notes": "string"
}
```

**Note:** `sellId` is required. All other fields optional. `status` defaults to `"pending"` if omitted.

**Expected Response 201:**
```json
{
  "message": "Shipment created successfully",
  "data": { "...full ShipmentResponse object..." }
}
```

**Frontend Impact:**
- Created from order detail page ("Create Shipment" button)
- Order's `shipments` array updated
- Order status may auto-update to "Processing" (backend-dependent)

---

## PATCH /shipments/:id/status
**Purpose:** Update shipment status and optionally add a tracking location/description
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric shipment ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "status": "in_transit | picked_up | out_for_delivery | delivered | failed | returned",
  "location": "Chicago, IL",
  "description": "Package picked up from sender"
}
```

**Note:** `status` is required. `location` and `description` are optional context for the status change.

**Expected Response 200:**
```json
{
  "message": "Shipment status updated",
  "data": { "...full ShipmentResponse object with updated status..." }
}
```

**Frontend Impact:**
- Status dropdown on shipment detail page
- Status change to `"delivered"` sets `deliveredAt` timestamp (backend)
- Order's `fulfillmentStatus` may update when all shipments delivered

---

## POST /shipments/:id/tracking
**Purpose:** Add a new tracking event to the shipment's history
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — numeric shipment ID

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "status": "in_transit",
  "location": "Chicago, IL",
  "description": "Package arrived at sorting facility",
  "eventTime": "2024-01-03T14:30:00Z"
}
```

**Note:** `status` and `description` are required. `location` and `eventTime` optional. If `eventTime` omitted, backend uses current timestamp.

**Expected Response 201:**
```json
{
  "message": "Tracking event added",
  "data": {
    "id": 5,
    "shipmentId": 1,
    "status": "in_transit",
    "location": "Chicago, IL",
    "description": "Package arrived at sorting facility",
    "eventTime": "2024-01-03T14:30:00Z",
    "createdAt": "2024-01-03T14:30:00Z"
  }
}
```

**Note:** Response `data` is a `TrackingEvent` object, NOT a `ShipmentResponse`. This is different from the status update endpoint.

**Frontend Impact:**
- Manual tracking event form on shipment detail page
- New event appended to tracking timeline
- Different from `PATCH /status` — this adds a history event without necessarily changing the current status
