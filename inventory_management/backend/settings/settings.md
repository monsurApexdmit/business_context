# Module Overview

**Module Name:** Settings
**Internal Identifier:** settings
**Purpose:** Stores all configurable store settings for a company as JSON blobs partitioned by category. Settings are auto-created with defaults on first access. Supports updating individual sections independently, and uploading logo and banner images.

---

# Business Context

Each company has a single settings record that holds all operational configuration for their store. Settings are organized into named sections (general, tax, shipping, payment, business, regional, notifications, store hours). This allows the frontend settings UI to load and save individual sections without affecting others. If a company has never saved settings, the system auto-creates the record with sensible defaults on the first GET request. Logo and banner images have dedicated upload endpoints that update the `logo_url` and `banner_url` fields on the settings record.

---

# Functional Scope

## Included

- Retrieve all settings sections in a single GET request
- Auto-create settings with defaults on first access
- Update any individual settings section independently (upsert behavior)
- Upload logo image (max 5MB)
- Upload banner image (max 10MB)
- All operations scoped to authenticated company via JWT

## Excluded

- Deleting settings (no DELETE endpoint)
- Resetting individual sections to defaults (no reset endpoint)
- Managing settings for multiple companies in a single request

---

# Architecture Notes

- Each company has exactly one row in the `settings` table.
- All update endpoints use **upsert**: if no settings record exists for the company, one is created with all defaults, then the requested section is updated.
- Each update endpoint modifies only a single JSON column (e.g., `general_settings`). Other sections in the same row are not touched.
- Each update endpoint returns only the updated section in its response, not the full settings object.
- `GET /settings/` returns all sections deserialized into named keys.
- `company_id` is extracted from the Bearer JWT and used as the lookup key for all operations.
- Image uploads store the file and update `settings.logo_url` or `settings.banner_url` in the same operation.
- `deleted_at` is present for soft-delete support, though the settings record is typically never deleted — only updated.

---

# Data Model

## Table: `settings`

| Column                  | Type      | Constraints / Notes                                         |
|-------------------------|-----------|-------------------------------------------------------------|
| `id`                    | uint      | Primary key, auto-increment; JSON key: `id`                 |
| `company_id`            | uint      | FK → `companies.id`; indexed; JSON key: `companyId`         |
| `general_settings`      | JSON      | `GeneralSettings` blob; JSON key: `generalSettings`         |
| `tax_settings`          | JSON      | `TaxSettings` blob; JSON key: `taxSettings`                 |
| `shipping_settings`     | JSON      | `ShippingSettings` blob; JSON key: `shippingSettings`       |
| `payment_settings`      | JSON      | `PaymentSettings` blob; JSON key: `paymentSettings`         |
| `business_settings`     | JSON      | `BusinessSettings` blob; JSON key: `businessSettings`       |
| `regional_settings`     | JSON      | `RegionalSettings` blob; JSON key: `regionalSettings`       |
| `notification_settings` | JSON      | `NotificationSettings` blob; JSON key: `notificationSettings`|
| `store_hours`           | JSON      | `StoreHoursData` blob; JSON key: `storeHours`               |
| `logo_url`              | string    | JSON key: `logoUrl`                                         |
| `banner_url`            | string    | JSON key: `bannerUrl`                                       |
| `uploaded_by`           | *uint     | FK → `saas_users.id`; nullable; last user to upload logo/banner; JSON key: `uploadedBy` |
| `created_at`            | timestamp | JSON key: `createdAt`                                       |
| `updated_at`            | timestamp | JSON key: `updatedAt`                                       |
| `deleted_at`            | timestamp | Soft delete                                                 |

## Relationships

None. Settings are self-contained per company.

---

# API Endpoints

| Method | Path                         | Auth       | Description                                       |
|--------|------------------------------|------------|---------------------------------------------------|
| GET    | `/settings/`                 | Bearer JWT | Retrieve all settings sections; auto-creates defaults if none exist |
| PUT    | `/settings/general`          | Bearer JWT | Update general settings section                   |
| PUT    | `/settings/tax`              | Bearer JWT | Update tax settings section                       |
| PUT    | `/settings/shipping`         | Bearer JWT | Update shipping settings section                  |
| PUT    | `/settings/payment`          | Bearer JWT | Update payment settings section                   |
| PUT    | `/settings/business`         | Bearer JWT | Update business settings section                  |
| PUT    | `/settings/regional`         | Bearer JWT | Update regional settings section                  |
| PUT    | `/settings/notifications`    | Bearer JWT | Update notification settings section              |
| PUT    | `/settings/store-hours`      | Bearer JWT | Update store hours section                        |
| POST   | `/settings/logo`             | Bearer JWT | Upload logo image (multipart)                     |
| POST   | `/settings/banner`           | Bearer JWT | Upload banner image (multipart)                   |

---

# Request Payloads

## PUT `/settings/general`

All fields optional.

```json
{
  "storeName": "My Store",
  "storeEmail": "store@example.com",
  "storePhone": "+1 234 567 8900",
  "storeAddress": "123 Main St, City, State",
  "storeDescription": "Welcome to our store"
}
```

## PUT `/settings/tax`

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

## PUT `/settings/shipping`

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

## PUT `/settings/payment`

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

## PUT `/settings/business`

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

## PUT `/settings/regional`

```json
{
  "language": "en-US",
  "currency": "USD",
  "timezone": "UTC"
}
```

## PUT `/settings/notifications`

```json
{
  "emailNotifications": true,
  "orderNotifications": true,
  "marketingEmails": false
}
```

## PUT `/settings/store-hours`

```json
{
  "monday":    {"open": "09:00", "close": "17:00", "isOpen": true},
  "tuesday":   {"open": "09:00", "close": "17:00", "isOpen": true},
  "wednesday": {"open": "09:00", "close": "17:00", "isOpen": true},
  "thursday":  {"open": "09:00", "close": "17:00", "isOpen": true},
  "friday":    {"open": "09:00", "close": "17:00", "isOpen": true},
  "saturday":  {"open": "10:00", "close": "15:00", "isOpen": true},
  "sunday":    {"open": "00:00", "close": "00:00", "isOpen": false}
}
```

## POST `/settings/logo`

**Content-Type:** `multipart/form-data`
- Field name: `file`
- Max size: 5MB
- Allowed types: `jpg`, `jpeg`, `png`, `gif`, `webp`
- Storage path: `uploads/logos/logo-{timestamp}.{ext}`

## POST `/settings/banner`

**Content-Type:** `multipart/form-data`
- Field name: `file`
- Max size: 10MB
- Allowed types: `jpg`, `jpeg`, `png`, `gif`, `webp`
- Storage path: `uploads/banners/banner-{timestamp}.{ext}`

---

# Response Contracts

## GET `/settings/` — 200 OK

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

**401:** `{"error": "Company ID not found in context"}`
**500:** `{"error": "Failed to create default settings"}`

## PUT `/settings/{section}` — 200 OK

Each section update returns only the updated section:

```json
{
  "message": "{Section} settings updated successfully",
  "data": { /* updated section object only */ }
}
```

**400:** `{"error": "Invalid request body", "details": "..."}`
**500:** `{"error": "Failed to update settings"}` or `{"error": "Failed to save settings"}`

## POST `/settings/logo` — 200 OK

```json
{
  "message": "Logo uploaded successfully",
  "data": {
    "logoUrl": "/uploads/logos/logo-1700000000.jpg"
  }
}
```

## POST `/settings/banner` — 200 OK

```json
{
  "message": "Banner uploaded successfully",
  "data": {
    "bannerUrl": "/uploads/banners/banner-1700000000.jpg"
  }
}
```

---

# Validation Rules

## Image Upload Constraints

| Endpoint            | Field | Max Size | Allowed Types                    |
|---------------------|-------|----------|----------------------------------|
| `POST /settings/logo`   | `file` | 5MB     | `jpg`, `jpeg`, `png`, `gif`, `webp` |
| `POST /settings/banner` | `file` | 10MB    | `jpg`, `jpeg`, `png`, `gif`, `webp` |

**Image upload error responses:**
- `400` — `{"error": "File is required"}`
- `400` — `{"error": "File size must not exceed 5MB"}` (or 10MB for banner)
- `400` — `{"error": "Invalid file type. Only jpg, jpeg, png, gif, webp are allowed"}`

All settings section update fields are optional. No field-level validation rules are documented beyond correct JSON structure.

---

# Business Logic Details

### Upsert Behavior
All PUT endpoints use upsert: if no settings record exists for the `company_id`, one is created with all default values first, then the specific section is updated. This ensures the record always exists after any update call.

### Default Values

| Section       | Field                      | Default          |
|---------------|----------------------------|------------------|
| Tax           | `defaultTaxRate`           | `10`             |
| Tax           | `taxInclusivePrice`        | `false`          |
| Tax           | `enableGSTTracking`        | `false`          |
| Tax           | `enableTaxExemption`       | `false`          |
| Tax           | `defaultShippingTax`       | `0`              |
| Shipping      | `enableShipping`           | `true`           |
| Shipping      | `defaultShippingCost`      | `5.99`           |
| Shipping      | `freeShippingThreshold`    | `50`             |
| Payment       | `enableCash`               | `true`           |
| Payment       | `enableCard`               | `true`           |
| Payment       | `enableOnline`             | `true`           |
| Payment       | `cardProcessingFee`        | `2.9`            |
| Regional      | `language`                 | `"en-US"`        |
| Regional      | `currency`                 | `"USD"`          |
| Regional      | `timezone`                 | `"UTC"`          |
| Notifications | `emailNotifications`       | `true`           |
| Notifications | `orderNotifications`       | `true`           |
| Notifications | `marketingEmails`          | `false`          |

### Auto-create on GET
`GET /settings/` auto-creates the settings record with all defaults if one does not exist for the company. No manual initialization step is needed.

### Partial Section Updates
Each PUT endpoint overwrites the entire JSON column for that section. To update a single field within a section, the caller must send the full section object with only that field changed.

---

# Dependencies

None. Settings are self-contained per company. No foreign key references to other business modules.

---

# Security / Permissions

- All endpoints require a valid Bearer JWT.
- `company_id` is extracted from the JWT. A user can only read or modify settings for their own company.
- `stripeKey` and `razorpayKey` are stored in `payment_settings`. These are sensitive fields. Their storage and access security (encryption at rest, audit logging) is not documented in the source.
- HTTP 401 is returned if `company_id` is not present in the JWT context.

---

# Error Handling

| HTTP Status | Trigger                                                                  |
|-------------|--------------------------------------------------------------------------|
| 400         | Invalid JSON request body; file not provided; file too large; invalid file type |
| 401         | `company_id` not found in JWT context                                    |
| 500         | Failed to create default settings; failed to update/save settings; failed to create upload directory; failed to save uploaded file; failed to update settings after upload |

---

# Testing Notes

- Verify auto-create: call `GET /settings/` for a company with no existing settings; confirm a record is created with correct defaults.
- Verify section isolation: update one section (e.g., tax) and confirm other sections (e.g., general) are unchanged.
- Verify upsert: call a PUT endpoint for a company with no settings record; confirm the record is created and the section is saved.
- Verify response from PUT returns only the updated section, not the full settings object.
- Test logo upload: valid file, oversized file (>5MB), disallowed file type — confirm correct HTTP 400 messages.
- Test banner upload: valid file, oversized file (>10MB), disallowed file type.
- Verify `logo_url` and `banner_url` are updated in the database after successful upload.
- Verify `GET /settings/` response includes `logoUrl` and `bannerUrl` fields if set.

---

# Open Questions / Missing Details

- Whether the old logo or banner file is deleted from disk when a new one is uploaded is not documented.
- Whether `stripeKey` and `razorpayKey` are encrypted at rest is not specified.
- The `GeneralSettings` fields (`storeName`, `storeEmail`, etc.) have no documented default values; it is assumed they default to empty strings.
- Whether the `shippingMethods` array in `ShippingSettings` has any default entries is not documented (assumed empty or a single default entry).
- Whether there is a maximum length on `open`/`close` time strings in `StoreHoursData` is not specified.
- Whether `GET /settings/` response includes `logoUrl` and `bannerUrl` at the top level or nested is not specified beyond them being columns on the `settings` row.

---

# Change Log / Notes

- Source documentation does not include a version history.
- This file was standardized from the original module notes on 2026-03-31.
