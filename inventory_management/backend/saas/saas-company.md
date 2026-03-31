# Module: SaaS Company

## Business Purpose
Manages the authenticated company's profile and company-level settings for SaaS users. Provides endpoints to get/update company info, get subscription status with user count, and manage company-specific settings (tax ID, currency, timezone, language).

---

## Database Tables

### Table: `companies`
| Column      | Type          | Notes                                                         |
|-------------|---------------|---------------------------------------------------------------|
| id          | uint (PK, AI) |                                                               |
| name        | string        | json: name                                                    |
| industry    | string        | json: industry                                                |
| website     | string        | json: website                                                 |
| phone       | string        | json: phone                                                   |
| address     | string        | json: address                                                 |
| city        | string        | json: city                                                    |
| state       | string        | json: state                                                   |
| zip_code    | string        | json: zipCode                                                 |
| country     | string        | json: country                                                 |
| logo        | string        | json: logo                                                    |
| description | string        | json: description                                             |
| status      | string        | 'trial', 'active', 'expired', 'suspended'; json: status       |
| created_at  | timestamp     | json: createdAt                                               |
| updated_at  | timestamp     | json: updatedAt                                               |

### Table: `company_settings`
| Column       | Type          | Notes                                            |
|--------------|---------------|--------------------------------------------------|
| id           | uint (PK, AI) |                                                  |
| company_id   | uint (FK)     | unique index; FK → companies.id; json: companyId |
| company_name | string        | json: companyName                                |
| tax_id       | string        | json: taxId                                      |
| tax_id_type  | string        | json: taxIdType                                  |
| tax_rate     | float64       | json: taxRate                                    |
| currency     | string        | default 'USD'; json: currency                    |
| timezone     | string        | default 'UTC'; json: timezone                    |
| language     | string        | default 'en'; json: language                     |
| created_at   | timestamp     | json: createdAt                                  |
| updated_at   | timestamp     | json: updatedAt                                  |

---

## Relationships
- Company ← SaasUsers (HasMany via company_id)
- Company ← Subscription (HasOne)
- Company → CompanySettings (HasOne)

---

## Business Logic
- `GetCompanyProfile` / `UpdateCompanyProfile` / `GetCompanyStatus` / `GetCompanySettings` / `UpdateCompanySettings`: all use `company_id` from JWT middleware context (or query param fallback).
- `GetCompanyStatus`: includes user count and max_users from subscription plan.
- `UpdateCompanyProfile`: partial update — only non-empty fields are updated.
- `UpdateCompanySettings`: upsert pattern via `FirstOrCreate` — creates record if none exists.
- `GetCompanySettings`: returns empty DTO (with zero values) if no record exists (no auto-create).

---

## Endpoints

### GET /auth/company/profile
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Company profile retrieved",
  "data": {
    "id": 1,
    "name": "Acme Corp",
    "industry": "Retail",
    "website": "https://acme.com",
    "phone": "+1234567890",
    "address": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "logo": "",
    "description": "",
    "status": "trial",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}`
- `404` — `{"message": "Company not found"}`

---

### PUT /auth/company/profile
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (UpdateCompanyProfileRequest):**
```json
{
  "companyProfile": {
    "name": "Acme Corp Updated",
    "industry": "E-Commerce",
    "website": "https://acme.com",
    "phone": "+1234567890",
    "address": "456 New St",
    "city": "Los Angeles",
    "state": "CA",
    "zipCode": "90001",
    "country": "US",
    "description": "Updated description"
  }
}
```

**Response 200:**
```json
{
  "message": "Company profile updated",
  "data": { /* CompanyDTO */ }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}` / `{"message": "Invalid request"}`
- `500` — `{"message": "Failed to update company"}`

---

### GET /auth/company/status
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Company status retrieved",
  "data": {
    "id": 1,
    "name": "Acme Corp",
    "status": "trial",
    "userCount": 3,
    "maxUsers": 5,
    "createdAt": "..."
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}`
- `404` — `{"message": "Company not found"}`

---

### GET /auth/company/settings
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Company settings retrieved",
  "data": {
    "id": 1,
    "companyId": 1,
    "companyName": "Acme Corp",
    "taxId": "US123456789",
    "taxIdType": "ein",
    "taxRate": 8.5,
    "currency": "USD",
    "timezone": "America/New_York",
    "language": "en",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

> If no settings record exists, returns zero-value DTO (id: 0, all strings empty).

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}`

---

### PUT /auth/company/settings
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body (UpdateCompanySettingsRequest — all optional):**
```json
{
  "companyName": "Acme Corp",
  "taxId": "US123456789",
  "taxIdType": "ein",
  "taxRate": 8.5,
  "currency": "USD",
  "timezone": "America/New_York",
  "language": "en"
}
```

**Response 200:**
```json
{
  "message": "Company settings updated",
  "data": { /* CompanySettingsDTO */ }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid or missing company ID"}` / `{"message": "Invalid request"}`
- `500` — `{"message": "Failed to update settings"}`

---

## Dependencies
- **saas-billing** — subscription plan provides maxUsers for status endpoint
- **saas-team** — user count is from saas_users table
