# Module Overview

**Module Name:** SaaS Company
**Scope:** Backend — SaaS Platform
**Purpose:** Manages the authenticated company's profile and company-level settings. Provides endpoints to retrieve and update company information, check subscription/user status, and manage company-specific configuration (tax ID, currency, timezone, language).

---

# Business Context

In a multi-tenant SaaS architecture, each company operates as an isolated tenant. This module gives company administrators control over their organization's public profile (name, industry, contact details) and operational settings (currency, tax configuration, locale preferences). The company status field reflects the subscription lifecycle state and is used by other modules to gate access. The status endpoint surfaces subscription and team capacity data in a single call for use by dashboards and admin UIs.

---

# Functional Scope

## Included

- Retrieve the authenticated company's profile
- Update the authenticated company's profile (partial update — only non-empty fields written)
- Retrieve company subscription status including current user count and plan user limit
- Retrieve company-level settings (tax, currency, timezone, language)
- Create or update company-level settings (upsert via `FirstOrCreate`)

## Excluded

- Company registration or onboarding (handled by auth/registration flows)
- Subscription plan management or billing operations (handled by the saas-billing module)
- Team member management (handled by the saas-team module)
- Logo upload mechanics (the `logo` field stores a reference string; the upload process is external)
- Deleting or deactivating a company

---

# Architecture Notes

- All endpoints derive `company_id` from the JWT middleware context, with a query parameter fallback.
- `UpdateCompanyProfile` uses a partial update strategy — only fields with non-empty values are applied to the database record.
- `UpdateCompanySettings` uses a GORM `FirstOrCreate` upsert pattern — creates the settings record if one does not yet exist for the company.
- `GetCompanySettings` returns a zero-value DTO if no settings record exists; it does not auto-create a record on read.
- `GetCompanyStatus` joins subscription plan data to surface `maxUsers` alongside the live `userCount` from `saas_users`.
- The `company_settings` table has a unique index on `company_id`, enforcing a one-to-one relationship with `companies`.

---

# Data Model

**Table: `companies`**

| Column        | Type          | Constraints    | Notes                                             |
|---------------|---------------|----------------|---------------------------------------------------|
| `id`          | uint          | PK, Auto Increment |                                               |
| `name`        | string        |                | JSON key: `name`                                  |
| `industry`    | string        |                | JSON key: `industry`                              |
| `website`     | string        |                | JSON key: `website`                               |
| `phone`       | string        |                | JSON key: `phone`                                 |
| `address`     | string        |                | JSON key: `address`                               |
| `city`        | string        |                | JSON key: `city`                                  |
| `state`       | string        |                | JSON key: `state`                                 |
| `zip_code`    | string        |                | JSON key: `zipCode`                               |
| `country`     | string        |                | JSON key: `country`                               |
| `logo`        | string        |                | JSON key: `logo`; file path or URL                |
| `uploaded_by` | *uint         | FK → `saas_users.id`; nullable | JSON key: `uploadedBy`; who uploaded the company logo |
| `description` | string        |                | JSON key: `description`                           |
| `status`      | string        |                | JSON key: `status`; values: `trial`, `active`, `expired`, `suspended` |
| `created_at`  | timestamp     | Auto-managed   | JSON key: `createdAt`                             |
| `updated_at`  | timestamp     | Auto-managed   | JSON key: `updatedAt`                             |
| `deleted_at`  | timestamp     |                | Soft delete                                       |

---

**Table: `company_settings`**

| Column         | Type     | Constraints                              | Notes                          |
|----------------|----------|------------------------------------------|--------------------------------|
| `id`           | uint     | PK, Auto Increment                       |                                |
| `company_id`   | uint     | FK → `companies.id`, Unique Index        | JSON key: `companyId`          |
| `company_name` | string   |                                          | JSON key: `companyName`        |
| `tax_id`       | string   |                                          | JSON key: `taxId`              |
| `tax_id_type`  | string   |                                          | JSON key: `taxIdType`          |
| `tax_rate`     | float64  |                                          | JSON key: `taxRate`            |
| `currency`     | string   | Default `'USD'`                          | JSON key: `currency`           |
| `timezone`     | string   | Default `'UTC'`                          | JSON key: `timezone`           |
| `language`     | string   | Default `'en'`                           | JSON key: `language`           |
| `created_at`   | timestamp| Auto-managed                             | JSON key: `createdAt`          |
| `updated_at`   | timestamp| Auto-managed                             | JSON key: `updatedAt`          |

**Relationships:**

| Relationship                   | Type    | Key          | Notes                                      |
|--------------------------------|---------|--------------|--------------------------------------------|
| Company ← SaasUsers            | HasMany | `company_id` | User count sourced from `saas_users` table |
| Company ← Subscription         | HasOne  | `company_id` | Subscription plan provides `maxUsers`      |
| Company → CompanySettings      | HasOne  | `company_id` | One settings record per company            |

---

# API Endpoints

| Method | URL                         | Auth       | Purpose                                    |
|--------|-----------------------------|------------|--------------------------------------------|
| GET    | `/auth/company/profile`     | Bearer JWT | Retrieve the company's profile             |
| PUT    | `/auth/company/profile`     | Bearer JWT | Update the company's profile               |
| GET    | `/auth/company/status`      | Bearer JWT | Retrieve company status with user counts   |
| GET    | `/auth/company/settings`    | Bearer JWT | Retrieve company-level settings            |
| PUT    | `/auth/company/settings`    | Bearer JWT | Create or update company-level settings    |

---

# Request Payloads

**PUT `/auth/company/profile` — Request Body (`UpdateCompanyProfileRequest`)**

The `companyProfile` wrapper object is required. All inner fields are optional (partial update).

```json
{
  "companyProfile": {
    "name":        "string (optional)",
    "industry":    "string (optional)",
    "website":     "string (optional)",
    "phone":       "string (optional)",
    "address":     "string (optional)",
    "city":        "string (optional)",
    "state":       "string (optional)",
    "zipCode":     "string (optional)",
    "country":     "string (optional)",
    "description": "string (optional)"
  }
}
```

---

**PUT `/auth/company/settings` — Request Body (`UpdateCompanySettingsRequest`)**

All fields are optional. Omitted fields retain their existing values (or column defaults on first creation).

```json
{
  "companyName": "string (optional)",
  "taxId":       "string (optional)",
  "taxIdType":   "string (optional)",
  "taxRate":     "float64 (optional)",
  "currency":    "string (optional)",
  "timezone":    "string (optional)",
  "language":    "string (optional)"
}
```

---

# Response Contracts

**GET `/auth/company/profile`** — 200 OK

```json
{
  "message": "Company profile retrieved",
  "data": {
    "id":          1,
    "name":        "Acme Corp",
    "industry":    "Retail",
    "website":     "https://acme.com",
    "phone":       "+1234567890",
    "address":     "123 Main St",
    "city":        "New York",
    "state":       "NY",
    "zipCode":     "10001",
    "country":     "US",
    "logo":        "",
    "description": "",
    "status":      "trial",
    "createdAt":   "2024-01-01T00:00:00Z",
    "updatedAt":   "2024-01-01T00:00:00Z"
  }
}
```

---

**PUT `/auth/company/profile`** — 200 OK

```json
{
  "message": "Company profile updated",
  "data": { }
}
```

`data` is a `CompanyDTO` with the same shape as the GET profile response.

---

**GET `/auth/company/status`** — 200 OK

```json
{
  "message": "Company status retrieved",
  "data": {
    "id":        1,
    "name":      "Acme Corp",
    "status":    "trial",
    "userCount": 3,
    "maxUsers":  5,
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

---

**GET `/auth/company/settings`** — 200 OK

```json
{
  "message": "Company settings retrieved",
  "data": {
    "id":          1,
    "companyId":   1,
    "companyName": "Acme Corp",
    "taxId":       "US123456789",
    "taxIdType":   "ein",
    "taxRate":     8.5,
    "currency":    "USD",
    "timezone":    "America/New_York",
    "language":    "en",
    "createdAt":   "2024-01-01T00:00:00Z",
    "updatedAt":   "2024-01-01T00:00:00Z"
  }
}
```

> If no settings record exists for the company, the endpoint returns 200 with a zero-value DTO (`id: 0`, all strings empty, `taxRate: 0`). No record is created on read.

---

**PUT `/auth/company/settings`** — 200 OK

```json
{
  "message": "Company settings updated",
  "data": { }
}
```

`data` is a `CompanySettingsDTO` with the same shape as the GET settings response.

---

# Validation Rules

| Field / Context        | Rule                                                                              |
|------------------------|-----------------------------------------------------------------------------------|
| `company_id` (all)     | Derived from JWT; returns 400 if invalid or missing                               |
| PUT profile body       | Request body must be valid JSON with a `companyProfile` wrapper; returns 400 if malformed |
| PUT settings body      | Request body must be valid JSON; returns 400 if malformed                         |
| `status` (companies)   | Managed externally; valid values: `trial`, `active`, `expired`, `suspended`       |

---

# Business Logic Details

### GET `/auth/company/status`

- Fetches the company record.
- Fetches the associated subscription to retrieve `max_users` from the linked plan.
- Counts active/invited records in `saas_users` for the company.
- Returns both `userCount` and `maxUsers` to allow the caller to determine capacity.

### PUT `/auth/company/profile` (Partial Update)

- Only fields with non-empty values in the request are written to the database.
- Fields not included in the request or sent as empty strings are ignored and the existing values are preserved.

### PUT `/auth/company/settings` (Upsert)

- Uses `FirstOrCreate` to find an existing `company_settings` record for the company or create one.
- Then applies the provided fields to the record.
- Column defaults (`currency = 'USD'`, `timezone = 'UTC'`, `language = 'en'`) apply on initial creation if values are not provided.

### GET `/auth/company/settings` (No Auto-Create)

- If no record exists in `company_settings` for the company, a zero-value DTO is returned. No database record is created as a side effect of a GET.

---

# Dependencies

| Module / Table    | Relationship                                                                      |
|-------------------|-----------------------------------------------------------------------------------|
| `saas-billing`    | Subscription plan provides `maxUsers` value used by the status endpoint           |
| `saas-team`       | Live user count sourced from the `saas_users` table                               |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is always resolved from the JWT context — a company administrator can only access and modify their own company's data.
- No role-based differentiation within this module is documented; it is assumed that any authenticated company user may call these endpoints.

---

# Error Handling

| HTTP Status | Endpoint(s)                         | Trigger                                                  |
|-------------|-------------------------------------|----------------------------------------------------------|
| 400         | All                                 | Invalid or missing `company_id` in JWT context           |
| 400         | PUT `/auth/company/profile`         | Malformed or unparseable request body                    |
| 400         | PUT `/auth/company/settings`        | Malformed or unparseable request body                    |
| 404         | GET `/auth/company/profile`         | Company not found                                        |
| 404         | GET `/auth/company/status`          | Company not found                                        |
| 500         | PUT `/auth/company/profile`         | Database error while updating company record             |
| 500         | PUT `/auth/company/settings`        | Database error while upserting settings record           |

**Error response shape:**

```json
{ "message": "Descriptive error message" }
```

---

# Testing Notes

- Verify that GET `/auth/company/profile` returns the correct company for the JWT's `company_id`.
- Verify partial update: sending only `name` in the PUT body updates `name` but leaves all other fields unchanged.
- Verify that GET `/auth/company/settings` returns a zero-value DTO when no settings record exists, and does not create a record.
- Verify that PUT `/auth/company/settings` creates a new record when one does not exist (upsert path), and updates an existing one on subsequent calls.
- Verify the status endpoint returns accurate `userCount` reflecting active team members and correct `maxUsers` from the subscription plan.
- Test with an expired or absent subscription on the status endpoint to confirm graceful fallback for `maxUsers`.
- Test that a company from Tenant A cannot access Tenant B's data by using Tenant A's JWT with Tenant B's `company_id`.

---

# Open Questions / Missing Details

- **Role-based access control:** It is not documented whether only owners/admins can call the PUT profile and PUT settings endpoints, or if all authenticated users within the company have access.
- **`logo` update:** The `logo` field is part of the company profile DTO but is not listed as an updatable field in the PUT profile body. It is unclear if it can be updated through this endpoint or requires a separate upload endpoint.
- **`maxUsers` fallback:** The behavior when no subscription exists for the status endpoint is not documented. It is unclear whether a default value is returned or a specific error is raised.
- **`status` field management:** The `companies.status` field (`trial`, `active`, `expired`, `suspended`) is not updated by any endpoint in this module. The process that transitions company status is not documented here.
- **`taxIdType` on settings:** Valid values are not enumerated in the settings schema (unlike `billing_contacts.tax_id_type`). Confirm whether any validation is applied.

---

# Change Log / Notes

| Date       | Note                                                                   |
|------------|------------------------------------------------------------------------|
| 2026-03-31 | Initial standardized documentation pass from source notes              |
