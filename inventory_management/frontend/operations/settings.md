# Frontend Module: Store Settings

## Pages
- `/dashboard/settings` — Settings overview / all settings
- `/dashboard/settings/general` — Store name, email, phone, address
- `/dashboard/settings/tax` — Tax rates, GST tracking, exemptions
- `/dashboard/settings/shipping` — Shipping methods, costs, free shipping threshold
- `/dashboard/settings/payment` — Payment method toggles, gateway keys
- `/dashboard/settings/business` — Business info, logo, banner, social links
- `/dashboard/settings/regional` — Language, currency, timezone
- `/dashboard/settings/notifications` — Email and marketing notification preferences
- `/dashboard/settings/store-hours` — Operating hours per day
- `/dashboard/settings/security` — Change password

## API Service
`lib/settingsApi.ts`

---

## GET /settings
**Purpose:** Fetch all settings sections in a single request
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "general": {
      "storeName": "string",
      "storeEmail": "string",
      "storePhone": "string",
      "storeAddress": "string",
      "storeDescription": "string"
    },
    "tax": {
      "id": 1,
      "defaultTaxRate": 10.0,
      "taxInclusivePrice": false,
      "enableGSTTracking": false,
      "gstNumber": "string",
      "enableTaxExemption": false,
      "defaultShippingTax": 0.0
    },
    "shipping": {
      "id": 1,
      "enableShipping": true,
      "defaultShippingCost": 5.00,
      "freeShippingThreshold": 100.00,
      "shippingMethods": [
        { "id": "standard", "name": "Standard", "cost": 5.00, "estimatedDays": 5, "isActive": true },
        { "id": "express", "name": "Express", "cost": 15.00, "estimatedDays": 2, "isActive": true }
      ]
    },
    "payment": {
      "id": 1,
      "enableCash": true,
      "enableCard": true,
      "enableOnline": false,
      "stripeKey": "pk_xxx",
      "razorpayKey": "rzp_xxx",
      "cardProcessingFee": 2.9
    },
    "business": {
      "id": 1,
      "businessName": "string",
      "businessType": "retail | wholesale | restaurant | service",
      "registrationNumber": "string",
      "gstNumber": "string",
      "logoUrl": "string",
      "bannerUrl": "string",
      "website": "string",
      "socialLinks": {
        "facebook": "string",
        "instagram": "string",
        "twitter": "string"
      }
    },
    "regional": {
      "language": "en",
      "currency": "USD",
      "timezone": "UTC"
    },
    "notifications": {
      "emailNotifications": true,
      "orderNotifications": true,
      "marketingEmails": false
    },
    "storeHours": {
      "monday": { "open": "09:00", "close": "18:00", "isOpen": true },
      "tuesday": { "open": "09:00", "close": "18:00", "isOpen": true },
      "wednesday": { "open": "09:00", "close": "18:00", "isOpen": true },
      "thursday": { "open": "09:00", "close": "18:00", "isOpen": true },
      "friday": { "open": "09:00", "close": "18:00", "isOpen": true },
      "saturday": { "open": "10:00", "close": "16:00", "isOpen": true },
      "sunday": { "open": "00:00", "close": "00:00", "isOpen": false }
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Settings overview page loads all sections at once
- Global currency/timezone/language settings applied to UI formatting

---

## GET /settings/general
**Purpose:** Fetch only the general store information
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "storeName": "My Store",
    "storeEmail": "store@example.com",
    "storePhone": "+1234567890",
    "storeAddress": "123 Main St",
    "storeDescription": "string"
  }
}
```

---

## PUT /settings/general
**Purpose:** Update general store information (full replace)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "storeName": "My Store",
  "storeEmail": "store@example.com",
  "storePhone": "+1234567890",
  "storeAddress": "123 Main St",
  "storeDescription": "string"
}
```

**Note:** Uses `PUT` (full replace) — all fields required. This is the only settings section using PUT instead of PATCH.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...GeneralSettings object..." }
}
```

---

## GET /settings/tax
**Purpose:** Fetch tax configuration
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "defaultTaxRate": 10.0,
    "taxInclusivePrice": false,
    "enableGSTTracking": false,
    "gstNumber": "string",
    "enableTaxExemption": false,
    "defaultShippingTax": 0.0
  }
}
```

---

## PATCH /settings/tax
**Purpose:** Update tax settings (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "defaultTaxRate": 12.5,
  "taxInclusivePrice": true,
  "enableGSTTracking": true,
  "gstNumber": "GST123456",
  "enableTaxExemption": false,
  "defaultShippingTax": 5.0
}
```

**Expected Response 200:**
```json
{ "message": "string", "data": { "...TaxSettings object..." } }
```

**Frontend Impact:**
- Tax rate change affects order calculation throughout the app
- `taxInclusivePrice: true` changes how prices are displayed

---

## GET /settings/shipping
**Purpose:** Fetch shipping configuration
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "enableShipping": true,
    "defaultShippingCost": 5.00,
    "freeShippingThreshold": 100.00,
    "shippingMethods": [
      { "id": "standard", "name": "Standard", "cost": 5.00, "estimatedDays": 5, "isActive": true }
    ]
  }
}
```

---

## PATCH /settings/shipping
**Purpose:** Update shipping settings (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "enableShipping": true,
  "defaultShippingCost": 7.00,
  "freeShippingThreshold": 75.00,
  "shippingMethods": [
    { "id": "standard", "name": "Standard", "cost": 7.00, "estimatedDays": 5, "isActive": true },
    { "id": "express", "name": "Express", "cost": 18.00, "estimatedDays": 1, "isActive": false }
  ]
}
```

**Expected Response 200:**
```json
{ "message": "string", "data": { "...ShippingSettings object..." } }
```

**Frontend Impact:**
- `shippingMethods` array rendered as editable list
- Active shipping methods available in order creation form
- `freeShippingThreshold` used for free shipping banner in checkout

---

## GET /settings/payment
**Purpose:** Fetch payment gateway configuration
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "enableCash": true,
    "enableCard": true,
    "enableOnline": false,
    "stripeKey": "pk_xxx",
    "razorpayKey": "rzp_xxx",
    "cardProcessingFee": 2.9
  }
}
```

---

## PATCH /settings/payment
**Purpose:** Update payment settings (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "enableCash": true,
  "enableCard": true,
  "enableOnline": true,
  "stripeKey": "pk_live_xxx",
  "razorpayKey": "rzp_live_xxx",
  "cardProcessingFee": 2.9
}
```

**Expected Response 200:**
```json
{ "message": "string", "data": { "...PaymentSettings object..." } }
```

**Frontend Impact:**
- Payment method options in order creation driven by `enableCash`, `enableCard`, `enableOnline`
- Stripe/Razorpay keys used in checkout integration

---

## GET /settings/business
**Purpose:** Fetch business profile and branding settings
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "businessName": "string",
    "businessType": "string",
    "registrationNumber": "string",
    "gstNumber": "string",
    "logoUrl": "https://cdn.example.com/logo.png",
    "bannerUrl": "https://cdn.example.com/banner.jpg",
    "website": "string",
    "socialLinks": {
      "facebook": "string",
      "instagram": "string",
      "twitter": "string"
    }
  }
}
```

---

## PATCH /settings/business
**Purpose:** Update business profile settings (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "businessName": "string",
  "businessType": "string",
  "registrationNumber": "string",
  "gstNumber": "string",
  "logoUrl": "string",
  "bannerUrl": "string",
  "website": "string",
  "socialLinks": {
    "facebook": "string",
    "instagram": "string",
    "twitter": "string"
  }
}
```

**Note:** `logoUrl` and `bannerUrl` are set via the upload endpoints below. This PATCH is for updating the URL after upload, or other business fields.

**Expected Response 200:**
```json
{ "message": "string", "data": { "...BusinessSettings object..." } }
```

---

## GET /settings/regional
**Purpose:** Fetch regional/locale settings
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "language": "en",
    "currency": "USD",
    "timezone": "America/New_York"
  }
}
```

---

## PATCH /settings/regional
**Purpose:** Update regional settings (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "language": "en",
  "currency": "USD",
  "timezone": "America/New_York"
}
```

**Expected Response 200:**
```json
{ "message": "string", "data": { "...RegionalSettings object..." } }
```

**Frontend Impact:**
- Currency change requires page refresh for all price displays to update
- Timezone affects all date/time formatting across the app
- Language may trigger i18n reload

---

## GET /settings/notifications
**Purpose:** Fetch notification preferences
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "emailNotifications": true,
    "orderNotifications": true,
    "marketingEmails": false
  }
}
```

---

## PATCH /settings/notifications
**Purpose:** Update notification preferences (partial update)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (all fields optional):**
```json
{
  "emailNotifications": true,
  "orderNotifications": true,
  "marketingEmails": false
}
```

**Expected Response 200:**
```json
{ "message": "string", "data": { "...NotificationSettings object..." } }
```

---

## GET /settings/store-hours
**Purpose:** Fetch store operating hours for each day of the week
**Auth:** Bearer JWT required

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "monday": { "open": "09:00", "close": "18:00", "isOpen": true },
    "tuesday": { "open": "09:00", "close": "18:00", "isOpen": true },
    "wednesday": { "open": "09:00", "close": "18:00", "isOpen": true },
    "thursday": { "open": "09:00", "close": "18:00", "isOpen": true },
    "friday": { "open": "09:00", "close": "18:00", "isOpen": true },
    "saturday": { "open": "10:00", "close": "16:00", "isOpen": true },
    "sunday": { "open": "00:00", "close": "00:00", "isOpen": false }
  }
}
```

---

## PATCH /settings/store-hours
**Purpose:** Update store hours (partial or full day map)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "monday": { "open": "08:00", "close": "20:00", "isOpen": true },
  "sunday": { "open": "11:00", "close": "17:00", "isOpen": true }
}
```

**Note:** Send only the days being updated. TypeScript type is `StoreHours` which is `{ [key: string]: { open: string; close: string; isOpen: boolean } }`.

**Expected Response 200:**
```json
{ "message": "string", "data": { "...StoreHours object (full week map)..." } }
```

---

## POST /settings/change-password
**Purpose:** Change the current user's password
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "currentPassword": "oldpassword123",
  "newPassword": "newpassword456",
  "confirmPassword": "newpassword456"
}
```

**Note:** All three fields are required. Frontend should validate `newPassword === confirmPassword` before calling.

**Expected Response 200:**
```json
{
  "message": "Password changed successfully"
}
```

**Note:** Response is just `{ message }` with no `data` field.

**Frontend Impact:**
- Password change form on security settings page
- After success, user may be required to log in again (backend-dependent)

---

## POST /settings/upload-logo
**Purpose:** Upload a new store/business logo image
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`

**Query Params:**
- `company_id` (auto-injected)

**Form Fields:**
| Field | Type | Notes |
|-------|------|-------|
| `file` | File | Required — the logo image file |

**Note:** FormData field name is exactly `file`. The Content-Type header is explicitly set to `multipart/form-data` in the request config (unlike coupon/product uploads which delete it and let the browser set it).

**Expected Response 200:**
```json
{
  "message": "Logo uploaded successfully",
  "data": {
    "logoUrl": "https://cdn.example.com/logos/company-1-logo.png"
  }
}
```

**Frontend Impact:**
- `logoUrl` stored and displayed in business settings
- Logo appears in dashboard header and printed documents
- After upload, PATCH `/settings/business` should be called to persist the URL if not done automatically by backend

---

## POST /settings/upload-banner
**Purpose:** Upload a store banner image
**Auth:** Bearer JWT required
**Content-Type:** `multipart/form-data`

**Query Params:**
- `company_id` (auto-injected)

**Form Fields:**
| Field | Type | Notes |
|-------|------|-------|
| `file` | File | Required — the banner image file |

**Note:** Same pattern as upload-logo. FormData field name is `file`.

**Expected Response 200:**
```json
{
  "message": "Banner uploaded successfully",
  "data": {
    "bannerUrl": "https://cdn.example.com/banners/company-1-banner.jpg"
  }
}
```

**Frontend Impact:**
- `bannerUrl` stored and displayed in storefront/business profile
- After upload, PATCH `/settings/business` should be called to persist the URL if not done automatically
