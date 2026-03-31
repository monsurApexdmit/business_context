# Frontend Module: SaaS Company Profile & Settings

## Pages
- `/dashboard/settings/company` — Company profile management (name, industry, logo, etc.)
- `/dashboard/settings/billing-contact` — Billing contact and tax ID information
- `/dashboard/settings/company-settings` — Tax rate, currency, timezone, language
- `/dashboard/settings/danger-zone` — Delete company account

## API Service
`lib/saasCompanyApi.ts`

---

## GET /company/profile
**Purpose:** Fetch the current company's profile data
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
    "name": "string",
    "industry": "string",
    "website": "string",
    "phone": "string",
    "address": "string",
    "city": "string",
    "state": "string",
    "zipCode": "string",
    "country": "string",
    "logo": "string",
    "description": "string",
    "status": "trial | active | expired | suspended",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Company name displayed in dashboard header and sidebar
- Company status drives trial banner visibility and feature gating

---

## PATCH /company/profile
**Purpose:** Update company profile fields
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "companyProfile": {
    "name": "string",
    "industry": "string",
    "website": "string",
    "phone": "string",
    "address": "string",
    "city": "string",
    "state": "string",
    "zipCode": "string",
    "country": "string",
    "logo": "string",
    "description": "string"
  }
}
```

**Note:** The entire payload is nested under the `companyProfile` key. All fields are optional (partial update).

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...CompanyProfile object..." }
}
```

**Frontend Impact:**
- Profile form re-renders with updated data
- Company name update reflects across the UI immediately

---

## GET /company/billing-contact
**Purpose:** Fetch billing contact details (address, tax ID)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "companyId": 1,
    "email": "string",
    "phone": "string",
    "address": "string",
    "city": "string",
    "state": "string",
    "zipCode": "string",
    "country": "string",
    "taxId": "string",
    "taxIdType": "vat | ein | gst | other",
    "isDefault": true,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Billing contact form pre-population
- Tax ID shown on invoices

---

## PATCH /company/billing-contact
**Purpose:** Update billing contact information
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "email": "string",
  "phone": "string",
  "address": "string",
  "city": "string",
  "state": "string",
  "zipCode": "string",
  "country": "string",
  "taxId": "string",
  "taxIdType": "vat | ein | gst | other"
}
```

**Note:** `email`, `phone`, `address`, `city`, `state`, `zipCode`, `country` are required. `taxId` and `taxIdType` are optional.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...BillingContact object..." }
}
```

**Frontend Impact:**
- Invoice generation uses updated billing contact details

---

## GET /company/status
**Purpose:** Get company subscription and license status, trial info, user counts
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "name": "string",
    "status": "trial | active | expired | suspended",
    "licenseKey": "string",
    "licenseType": "trial | paid",
    "trialStartDate": "2024-01-01T00:00:00Z",
    "trialEndDate": "2024-01-14T00:00:00Z",
    "trialDaysRemaining": 7,
    "subscriptionId": 1,
    "subscriptionPlanId": 2,
    "subscriptionPlanName": "string",
    "subscriptionEndDate": "2025-01-01T00:00:00Z",
    "userCount": 3,
    "maxUsers": 10,
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Trial banner shows `trialDaysRemaining`
- Feature gates check `status` (expired/suspended blocks access)
- User invite button checks `userCount < maxUsers`

---

## GET /company/settings
**Purpose:** Fetch company operational settings (tax, currency, timezone, language)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "id": 1,
    "companyId": 1,
    "companyName": "string",
    "taxId": "string",
    "taxIdType": "string",
    "taxRate": 10.0,
    "currency": "USD",
    "timezone": "UTC",
    "language": "en",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Currency symbol displayed throughout pricing UI
- Timezone used for date formatting
- Tax rate applied in order totals

---

## PATCH /company/settings
**Purpose:** Update company operational settings
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "companyName": "string",
  "taxId": "string",
  "taxIdType": "vat | ein | gst | other",
  "taxRate": 10.0,
  "currency": "USD",
  "timezone": "UTC",
  "language": "en"
}
```

**Note:** All fields optional for partial update.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": { "...CompanySettings object..." }
}
```

**Frontend Impact:**
- Currency/timezone changes require page refresh to propagate through all UI components

---

## DELETE /company
**Purpose:** Permanently delete the company and all associated data
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body (sent as DELETE body):**
```json
{
  "confirmPassword": "string",
  "reason": "string"
}
```

**Note:** `reason` is optional. The axios call uses `{ data: payload }` to send a body with DELETE.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "companyId": 1,
    "success": true,
    "deletedAt": "2024-01-01T00:00:00Z",
    "dataRetentionDays": 30
  }
}
```

**Frontend Impact:**
- On success, clears localStorage and redirects to `/auth/login`
- Irreversible — destroys all company data, users, products, orders
