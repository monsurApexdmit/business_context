# Module: Settings

## Business Purpose
Stores all configurable store settings for a company as JSON blobs partitioned by category. Settings are auto-created with defaults on first access. Supports updating individual sections (general, tax, shipping, payment, business, regional, notifications, store hours) and uploading logo/banner images.

---

## Database Table: `settings`

| Column               | Type          | Notes                                                   |
|----------------------|---------------|---------------------------------------------------------|
| id                   | uint (PK, AI) | json: id                                                |
| company_id           | uint (FK)     | FK → companies.id; indexed; json: companyId             |
| general_settings     | JSON          | GeneralSettings blob; json: generalSettings             |
| tax_settings         | JSON          | TaxSettings blob; json: taxSettings                     |
| shipping_settings    | JSON          | ShippingSettings blob; json: shippingSettings           |
| payment_settings     | JSON          | PaymentSettings blob; json: paymentSettings             |
| business_settings    | JSON          | BusinessSettings blob; json: businessSettings           |
| regional_settings    | JSON          | RegionalSettings blob; json: regionalSettings           |
| notification_settings| JSON          | NotificationSettings blob; json: notificationSettings   |
| store_hours          | JSON          | StoreHoursData blob; json: storeHours                   |
| logo_url             | string        | json: logoUrl                                           |
| banner_url           | string        | json: bannerUrl                                         |
| created_at           | timestamp     | json: createdAt                                         |
| updated_at           | timestamp     | json: updatedAt                                         |

---

## Settings Schemas

### GeneralSettings
```json
{
  "storeName": "My Store",
  "storeEmail": "store@example.com",
  "storePhone": "+1 234 567 8900",
  "storeAddress": "123 Main St, City, State",
  "storeDescription": "Welcome to our store"
}
```

### TaxSettings
```json
{
  "defaultTaxRate": 10,
  "taxInclusivePrice": false,
  "enableGSTTracking": false,
  "gstNumber": "",
  "enableTaxExemption": false,
  "defaultShippingTax": 0
}
```

### ShippingSettings
```json
{
  "enableShipping": true,
  "defaultShippingCost": 5.99,
  "freeShippingThreshold": 50,
  "shippingMethods": [
    {
      "id": "standard",
      "name": "Standard Shipping",
      "cost": 5.99,
      "estimatedDays": 5,
      "isActive": true
    }
  ]
}
```

### PaymentSettings
```json
{
  "enableCash": true,
  "enableCard": true,
  "enableOnline": true,
  "cardProcessingFee": 2.9,
  "stripeKey": "",
  "razorpayKey": ""
}
```

### BusinessSettings
```json
{
  "businessName": "My Business",
  "businessType": "Retail",
  "registrationNumber": "",
  "gstNumber": "",
  "website": "https://example.com",
  "socialLinks": {
    "facebook": "",
    "instagram": "",
    "twitter": ""
  }
}
```

### RegionalSettings
```json
{
  "language": "en-US",
  "currency": "USD",
  "timezone": "UTC"
}
```

### NotificationSettings
```json
{
  "emailNotifications": true,
  "orderNotifications": true,
  "marketingEmails": false
}
```

### StoreHoursData
```json
{
  "monday":    { "open": "09:00", "close": "17:00", "isOpen": true },
  "tuesday":   { "open": "09:00", "close": "17:00", "isOpen": true },
  "wednesday": { "open": "09:00", "close": "17:00", "isOpen": true },
  "thursday":  { "open": "09:00", "close": "17:00", "isOpen": true },
  "friday":    { "open": "09:00", "close": "17:00", "isOpen": true },
  "saturday":  { "open": "10:00", "close": "15:00", "isOpen": true },
  "sunday":    { "open": "00:00", "close": "00:00", "isOpen": false }
}
```

---

## Business Logic
- All update endpoints use **upsert**: if no settings record exists for the company, one is created with all defaults first, then the requested section is updated.
- Each update endpoint updates only a single JSON column (e.g., `general_settings`), not the whole row.
- The response from each update endpoint returns the updated section only (not the full settings).
- `GetSettings` returns the full settings object deserialized into its sections.
- Scoped by `company_id` from JWT context.

---

## Endpoints

### GET /settings/
**Auth:** Bearer JWT required
**Purpose:** Retrieve all settings sections. Auto-creates with defaults if none exist.

**Response 200:**
```json
{
  "message": "Settings retrieved successfully",
  "data": {
    "general":       { /* GeneralSettings */ },
    "tax":           { /* TaxSettings */ },
    "shipping":      { /* ShippingSettings */ },
    "payment":       { /* PaymentSettings */ },
    "business":      { /* BusinessSettings */ },
    "regional":      { /* RegionalSettings */ },
    "notifications": { /* NotificationSettings */ },
    "storeHours":    { /* StoreHoursData */ }
  }
}
```

**Error Responses:**
- `401` — `{"error": "Company ID not found in context"}`
- `500` — `{"error": "Failed to create default settings"}`

---

### PUT /settings/general
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** GeneralSettings object (all fields optional).

**Response 200:**
```json
{
  "message": "General settings updated successfully",
  "data": { /* GeneralSettings object */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid request body", "details": "..."}`
- `500` — `{"error": "Failed to update settings"}` / `{"error": "Failed to save settings"}`

---

### PUT /settings/tax
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** TaxSettings object.

**Response 200:**
```json
{
  "message": "Tax settings updated successfully",
  "data": { /* TaxSettings object */ }
}
```

---

### PUT /settings/shipping
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** ShippingSettings object.

**Response 200:**
```json
{
  "message": "Shipping settings updated successfully",
  "data": { /* ShippingSettings object */ }
}
```

---

### PUT /settings/payment
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** PaymentSettings object.

**Response 200:**
```json
{
  "message": "Payment settings updated successfully",
  "data": { /* PaymentSettings object */ }
}
```

---

### PUT /settings/business
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** BusinessSettings object.

**Response 200:**
```json
{
  "message": "Business settings updated successfully",
  "data": { /* BusinessSettings object */ }
}
```

---

### PUT /settings/regional
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** RegionalSettings object.

**Response 200:**
```json
{
  "message": "Regional settings updated successfully",
  "data": { /* RegionalSettings object */ }
}
```

---

### PUT /settings/notifications
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** NotificationSettings object.

**Response 200:**
```json
{
  "message": "Notification settings updated successfully",
  "data": { /* NotificationSettings object */ }
}
```

---

### PUT /settings/store-hours
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** StoreHoursData object.

**Response 200:**
```json
{
  "message": "Store hours updated successfully",
  "data": { /* StoreHoursData object */ }
}
```

---

### POST /settings/logo
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Field:** `file`
**Constraints:** max 5MB; allowed types: jpg, jpeg, png, gif, webp
**Storage:** `uploads/logos/logo-{timestamp}.{ext}`

**Response 200:**
```json
{
  "message": "Logo uploaded successfully",
  "data": {
    "logoUrl": "/uploads/logos/logo-1700000000.jpg"
  }
}
```

**Error Responses:**
- `400` — `{"error": "File is required"}` / `{"error": "File size must not exceed 5MB"}` / `{"error": "Invalid file type. Only jpg, jpeg, png, gif, webp are allowed"}`
- `500` — `{"error": "Failed to create upload directory"}` / `{"error": "Failed to save file"}` / `{"error": "Failed to update settings"}`

---

### POST /settings/banner
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`
**Field:** `file`
**Constraints:** max 10MB; allowed types: jpg, jpeg, png, gif, webp
**Storage:** `uploads/banners/banner-{timestamp}.{ext}`

**Response 200:**
```json
{
  "message": "Banner uploaded successfully",
  "data": {
    "bannerUrl": "/uploads/banners/banner-1700000000.jpg"
  }
}
```

**Error Responses:** Same pattern as UploadLogo but for banners (max 10MB).

---

## Dependencies
- None (settings are self-contained per company)
