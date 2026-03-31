# Module Overview

**Module Name:** SaaS Billing
**Scope:** Backend — SaaS Platform
**Purpose:** Manages subscription plans, company subscriptions, payment history, and billing contact information. Handles the full subscription lifecycle: viewing, renewing, cancelling, and upgrading. Also manages billing contact details used for invoicing.

---

# Business Context

Each company on the SaaS platform subscribes to a plan that governs feature limits (max users, max products, max branches). This module provides the data layer and business rules for subscription management. Payment history is recorded for auditing and invoice retrieval. Billing contact information is maintained separately from the company profile to support invoicing workflows. A test/seeding endpoint is available for creating trial subscriptions during development.

Stripe integration fields (`stripe_subscription_id`, `stripe_payment_id`) are present in the schema, indicating the platform intends to use or already uses Stripe for payment processing — however, the Stripe webhook and payment processing logic is not documented here.

---

# Functional Scope

## Included

- Listing all active subscription plans
- Retrieving the current company subscription (regardless of status)
- Retrieving payment history for the company
- Renewing a subscription (extending period by 1 month or 1 year based on billing period)
- Cancelling a subscription
- Upgrading a subscription to a new plan (requires an active subscription)
- Retrieving the billing contact record
- Creating or updating the billing contact record (upsert)
- Creating a test/seed subscription (trial, 10-day duration)

## Excluded

- Payment processing or Stripe webhook handling (Stripe fields are stored but processing is not documented here)
- Plan creation or administration (plans are seeded/managed outside this module)
- Invoice generation (invoice URL is stored but generation is external)
- Automated renewal or expiry (no background job is documented)
- Downgrading a subscription

---

# Architecture Notes

- `company_id` is derived from the JWT middleware context on all endpoints.
- `GetCurrentSubscription` retrieves the most recently created subscription (`ORDER BY created_at DESC`) regardless of its `status` — expired and cancelled subscriptions can be returned.
- `GetPaymentHistory` orders results by `payment_date DESC`.
- `RenewSubscription` extends from `current_period_end` (not from now), adding 1 month or 1 year based on the plan's `billing_period`.
- `UpgradeSubscription` requires the current subscription to have `status = 'active'`.
- `UpdateBillingContact` uses a `FirstOrCreate` upsert — one billing contact record per company (unique index on `company_id`).
- `CreateSubscriptionForCompany` (testing/seeding) creates a 10-day subscription at `plan_id = 1` and will return 409 if a subscription already exists.
- `price` and `amount` fields are stored as integers in cents (e.g., `2999` = $29.99).
- `features` in `subscription_plans` is stored as a JSON-serialized string, not a native JSON column.

---

# Data Model

**Table: `subscription_plans`**

| Column           | Type     | Constraints          | Notes                                                       |
|------------------|----------|----------------------|-------------------------------------------------------------|
| `id`             | uint     | PK, Auto Increment   |                                                             |
| `name`           | string   |                      | JSON key: `name`                                            |
| `description`    | string   |                      | JSON key: `description`                                     |
| `price`          | int64    |                      | JSON key: `price`; stored in cents                          |
| `billing_period` | string   |                      | JSON key: `billingPeriod`; values: `monthly`, `yearly`      |
| `max_users`      | int      |                      | JSON key: `maxUsers`                                        |
| `max_products`   | int      |                      | JSON key: `maxProducts`                                     |
| `max_branches`   | int      |                      | JSON key: `maxBranches`                                     |
| `features`       | string   |                      | JSON key: `features`; JSON-serialized array string          |
| `is_active`      | bool     |                      | JSON key: `isActive`; only active plans returned by GetPlans |
| `is_featured`    | bool     |                      | JSON key: `isFeatured`                                      |
| `created_at`     | timestamp| Auto-managed         | JSON key: `createdAt`                                       |
| `updated_at`     | timestamp| Auto-managed         |                                                             |

---

**Table: `subscriptions`**

| Column                   | Type             | Constraints          | Notes                                                   |
|--------------------------|------------------|----------------------|---------------------------------------------------------|
| `id`                     | uint             | PK, Auto Increment   |                                                         |
| `company_id`             | uint             | FK → `companies.id`  |                                                         |
| `plan_id`                | uint             | FK → `subscription_plans.id` |                                                 |
| `status`                 | string           |                      | JSON key: `status`; values: `active`, `expired`, `cancelled`, `pending` |
| `current_period_start`   | timestamp        |                      | JSON key: `currentPeriodStart`                          |
| `current_period_end`     | timestamp        |                      | JSON key: `currentPeriodEnd`                            |
| `next_billing_date`      | timestamp        |                      | JSON key: `nextBillingDate`                             |
| `auto_renew`             | bool             |                      | JSON key: `autoRenew`                                   |
| `stripe_subscription_id` | string           |                      | JSON key: `stripeSubscriptionId`                        |
| `cancelled_at`           | *timestamp       | Nullable             | JSON key: `cancelledAt`                                 |
| `created_at`             | timestamp        | Auto-managed         | JSON key: `createdAt`                                   |
| `updated_at`             | timestamp        | Auto-managed         |                                                         |

---

**Table: `payments`**

| Column              | Type     | Constraints                | Notes                                              |
|---------------------|----------|----------------------------|----------------------------------------------------|
| `id`                | uint     | PK, Auto Increment         |                                                    |
| `subscription_id`   | uint     | FK → `subscriptions.id`    |                                                    |
| `company_id`        | uint     | FK → `companies.id`        |                                                    |
| `amount`            | int64    |                            | JSON key: `amount`; in cents                       |
| `status`            | string   |                            | JSON key: `status`; values: `paid`, `pending`, `failed` |
| `payment_method`    | string   |                            | JSON key: `paymentMethod`                          |
| `payment_date`      | timestamp|                            | JSON key: `paymentDate`; default ordering DESC     |
| `invoice_number`    | string   |                            | JSON key: `invoiceNumber`                          |
| `invoice_url`       | string   |                            | JSON key: `invoiceUrl`                             |
| `stripe_payment_id` | string   |                            | JSON key: `stripePaymentId`                        |
| `description`       | string   |                            | JSON key: `description`                            |
| `created_at`        | timestamp| Auto-managed               | JSON key: `createdAt`                              |
| `updated_at`        | timestamp| Auto-managed               |                                                    |

---

**Table: `billing_contacts`**

| Column        | Type     | Constraints              | Notes                                                  |
|---------------|----------|--------------------------|--------------------------------------------------------|
| `id`          | uint     | PK, Auto Increment       |                                                        |
| `company_id`  | uint     | Unique Index             | JSON key: `companyId`; one record per company          |
| `email`       | string   |                          | JSON key: `email`                                      |
| `phone`       | string   |                          | JSON key: `phone`                                      |
| `address`     | string   |                          | JSON key: `address`                                    |
| `city`        | string   |                          | JSON key: `city`                                       |
| `state`       | string   |                          | JSON key: `state`                                      |
| `zip_code`    | string   |                          | JSON key: `zipCode`                                    |
| `country`     | string   |                          | JSON key: `country`                                    |
| `tax_id`      | string   |                          | JSON key: `taxId`                                      |
| `tax_id_type` | string   |                          | JSON key: `taxIdType`; values: `vat`, `ein`, `gst`, `other` |
| `is_default`  | bool     |                          | JSON key: `isDefault`                                  |
| `created_at`  | timestamp| Auto-managed             | JSON key: `createdAt`                                  |
| `updated_at`  | timestamp| Auto-managed             | JSON key: `updatedAt`                                  |

**Relationships:**

| Relationship                        | Type      | Key              | Notes                                          |
|-------------------------------------|-----------|------------------|------------------------------------------------|
| Subscription → SubscriptionPlan     | BelongsTo | `plan_id`        | Preloaded on `GetCurrentSubscription`          |
| Subscription ← Payments             | HasMany   | `subscription_id`|                                                |
| Company → Subscription              | HasOne    | `company_id`     |                                                |
| Company → BillingContact            | HasOne    | `company_id`     |                                                |

---

# API Endpoints

| Method | URL                           | Auth       | Purpose                                           |
|--------|-------------------------------|------------|---------------------------------------------------|
| GET    | `/billing/plans`              | Bearer JWT | List all active subscription plans                |
| GET    | `/billing/subscription`       | Bearer JWT | Get the company's current subscription            |
| GET    | `/billing/payments`           | Bearer JWT | Get the company's payment history                 |
| POST   | `/billing/renew`              | Bearer JWT | Renew a subscription                              |
| POST   | `/billing/cancel`             | Bearer JWT | Cancel a subscription                             |
| POST   | `/billing/upgrade`            | Bearer JWT | Upgrade a subscription to a new plan              |
| POST   | `/billing/create-subscription`| Bearer JWT | Create a trial subscription (testing/seeding only)|
| GET    | `/billing/contact`            | Bearer JWT | Get the billing contact for the company           |
| PUT    | `/billing/contact`            | Bearer JWT | Create or update the billing contact (upsert)     |

---

# Request Payloads

**POST `/billing/renew` — Request Body**

```json
{
  "subscriptionId": "uint (required)",
  "autoRenew":      "bool (optional)"
}
```

---

**POST `/billing/cancel` — Request Body**

```json
{
  "subscriptionId": "uint (required)"
}
```

---

**POST `/billing/upgrade` — Request Body**

```json
{
  "newPlanId": "uint (required)"
}
```

---

**POST `/billing/create-subscription` — Request Body**

```json
{
  "companyId": "uint (required)",
  "planId":    "uint (required)"
}
```

---

**PUT `/billing/contact` — Request Body (`BillingContactRequest`, all fields optional)**

```json
{
  "email":      "string (optional)",
  "phone":      "string (optional)",
  "address":    "string (optional)",
  "city":       "string (optional)",
  "state":      "string (optional)",
  "zipCode":    "string (optional)",
  "country":    "string (optional)",
  "taxId":      "string (optional)",
  "taxIdType":  "string (optional) — vat | ein | gst | other"
}
```

---

# Response Contracts

**GET `/billing/plans`** — 200 OK

```json
{
  "message": "Plans retrieved",
  "data": [
    {
      "id":            1,
      "name":          "Trial",
      "description":   "10-day free trial",
      "price":         0,
      "billingPeriod": "monthly",
      "maxUsers":      5,
      "maxProducts":   100,
      "maxBranches":   1,
      "features":      "[\"Feature 1\",\"Feature 2\"]",
      "isActive":      true,
      "isFeatured":    false,
      "createdAt":     "2024-01-01T00:00:00Z"
    }
  ]
}
```

---

**GET `/billing/subscription`** — 200 OK

```json
{
  "message": "Subscription retrieved",
  "data": {
    "id":                 1,
    "planId":             1,
    "planName":           "Trial",
    "price":              0,
    "billingPeriod":      "monthly",
    "status":             "active",
    "currentPeriodStart": "2024-01-01T00:00:00Z",
    "currentPeriodEnd":   "2024-01-11T00:00:00Z",
    "nextBillingDate":    "2024-01-11T00:00:00Z",
    "autoRenew":          true,
    "createdAt":          "2024-01-01T00:00:00Z"
  }
}
```

---

**GET `/billing/payments`** — 200 OK

```json
{
  "message": "Payment history retrieved",
  "data": {
    "payments": [
      {
        "id":            1,
        "paymentDate":   "2024-01-01T00:00:00Z",
        "createdAt":     "2024-01-01T00:00:00Z",
        "invoiceNumber": "INV-2024-001",
        "amount":        2999,
        "status":        "paid",
        "invoiceUrl":    ""
      }
    ]
  }
}
```

---

**POST `/billing/renew`** — 200 OK

```json
{
  "message": "Subscription renewed successfully",
  "data": {
    "currentPeriodEnd": "2024-02-11T00:00:00Z",
    "nextBillingDate":  "2024-02-11T00:00:00Z",
    "status":           "active"
  }
}
```

---

**POST `/billing/cancel`** — 200 OK

```json
{
  "message": "Subscription cancelled successfully",
  "data": {
    "status":      "cancelled",
    "cancelledAt": "2024-01-15T00:00:00Z"
  }
}
```

---

**POST `/billing/upgrade`** — 200 OK

```json
{
  "message": "Subscription upgraded successfully",
  "data": {
    "newPlanId":   2,
    "newPlanName": "Professional"
  }
}
```

---

**POST `/billing/create-subscription`** — 201 Created

```json
{
  "message": "Subscription created successfully",
  "data": {
    "id":                 1,
    "companyId":          1,
    "planId":             1,
    "status":             "active",
    "currentPeriodStart": "2024-01-01T00:00:00Z",
    "currentPeriodEnd":   "2024-01-11T00:00:00Z",
    "nextBillingDate":    "2024-01-11T00:00:00Z",
    "autoRenew":          true
  }
}
```

---

**GET `/billing/contact`** — 200 OK

```json
{
  "message": "Billing contact retrieved",
  "data": {
    "id":         1,
    "companyId":  1,
    "email":      "billing@acme.com",
    "phone":      "+1234567890",
    "address":    "123 Main St",
    "city":       "New York",
    "state":      "NY",
    "zipCode":    "10001",
    "country":    "US",
    "taxId":      "US123456789",
    "taxIdType":  "ein",
    "isDefault":  true,
    "createdAt":  "2024-01-01T00:00:00Z",
    "updatedAt":  "2024-01-01T00:00:00Z"
  }
}
```

---

**PUT `/billing/contact`** — 200 OK

```json
{
  "message": "Billing contact updated",
  "data": { }
}
```

`data` is a `BillingContactDTO` with the same shape as the GET contact response.

---

# Validation Rules

| Field / Context             | Rule                                                                              |
|-----------------------------|-----------------------------------------------------------------------------------|
| `company_id` (all)          | Derived from JWT; 401 if missing                                                  |
| `subscriptionId` (renew)    | Required; subscription must exist — 404 if not found                             |
| `subscriptionId` (cancel)   | Required; subscription must exist — 404 if not found                             |
| `newPlanId` (upgrade)       | Required; plan must exist — 404 if not found; subscription must be `active` — 404 if not |
| `companyId` (create-subscription) | Required; company must exist — 404 if not found                          |
| `planId` (create-subscription)    | Required; plan must exist — 404 if not found                             |
| `taxIdType` (billing contact)     | If provided, must be one of: `vat`, `ein`, `gst`, `other`               |

---

# Business Logic Details

### GetPlans

- Returns only plans where `is_active = true`. Inactive plans are hidden from the listing.

### GetCurrentSubscription

- Fetches the most recently created subscription for the company (`ORDER BY created_at DESC`).
- Returns the subscription regardless of its `status` (active, expired, cancelled, pending).
- Preloads the associated `SubscriptionPlan`.

### RenewSubscription

- Extends the subscription period starting from `current_period_end` (not from the current date).
- Extension duration: +1 month if `billing_period = 'monthly'`; +1 year if `billing_period = 'yearly'`.
- Sets `status = 'active'`.
- Updates `next_billing_date` to match the new `current_period_end`.
- Applies the `autoRenew` flag from the request body if provided.

### CancelSubscription

- Sets `status = 'cancelled'`.
- Records the cancellation timestamp in `cancelled_at`.
- Does not delete or modify the subscription period dates.

### UpgradeSubscription

- Requires the company's current subscription to have `status = 'active'` — returns 404 if no active subscription is found.
- Updates `plan_id` on the existing subscription record to `newPlanId`.
- Does not change the current billing period dates.

### CreateSubscriptionForCompany (Testing / Seeding Only)

- Verifies the company and plan both exist.
- Returns 409 if the company already has a subscription record.
- Creates a subscription with:
  - `status = 'active'`
  - `current_period_start = now`
  - `current_period_end = now + 10 days`
  - `next_billing_date = now + 10 days`
- Plan ID is passed in the request (the source notes indicate it defaults to plan_id 1 in the seeding context).

### UpdateBillingContact (Upsert)

- Uses `FirstOrCreate` on `company_id` to find an existing `billing_contacts` record or create one.
- Then applies the provided fields to the record.
- One billing contact per company is enforced by the unique index on `company_id`.

---

# Dependencies

| Module / Table  | Relationship                                                                              |
|-----------------|-------------------------------------------------------------------------------------------|
| `saas-team`     | `max_users` from subscription plan enforces team size limits in the saas-team module      |
| `saas-company`  | Company `status` field reflects subscription state; company existence is a prerequisite   |
| `companies`     | `company_id` FK on subscriptions, payments, and billing_contacts                          |

---

# Security / Permissions

- All endpoints require a valid Bearer JWT token.
- `company_id` is resolved from the JWT — a company's billing data is not accessible to other companies.
- `POST /billing/create-subscription` is intended for testing/seeding only and should be protected or removed in production environments. There is no documented access restriction beyond Bearer JWT.
- Stripe IDs (`stripe_subscription_id`, `stripe_payment_id`) are stored in the database but no Stripe API keys or webhook secrets are handled in this module's documentation.

---

# Error Handling

| HTTP Status | Endpoint                         | Trigger                                                                   |
|-------------|----------------------------------|---------------------------------------------------------------------------|
| 400         | POST endpoints                   | Malformed or missing required fields in request body                      |
| 401         | All                              | Missing or invalid `company_id` in JWT token                              |
| 404         | GET `/billing/subscription`      | No subscription record found for the company                              |
| 404         | GET `/billing/contact`           | No billing contact record found for the company                           |
| 404         | POST `/billing/renew`            | Subscription not found                                                    |
| 404         | POST `/billing/cancel`           | Subscription not found                                                    |
| 404         | POST `/billing/upgrade`          | No active subscription found; or new plan not found                       |
| 404         | POST `/billing/create-subscription` | Company not found; or plan not found                                   |
| 409         | POST `/billing/create-subscription` | Subscription already exists for this company                           |
| 500         | All                              | Internal database or server errors                                        |

**Error response shape:**

```json
{ "message": "Descriptive error message" }
```

With optional `error` field for detail:

```json
{ "message": "Invalid request", "error": "..." }
```

---

# Testing Notes

- Verify that GET `/billing/plans` returns only plans where `is_active = true`.
- Verify that GET `/billing/subscription` returns the most recently created subscription and preloads plan data.
- Verify that GET `/billing/subscription` returns a subscription even when its status is `cancelled` or `expired`.
- Test renew: verify `current_period_end` and `next_billing_date` are extended from the old `current_period_end`, not from the current date; verify monthly vs. yearly period extension.
- Test cancel: verify `status = 'cancelled'` and `cancelled_at` is set; verify billing period dates are unchanged.
- Test upgrade: verify it requires `status = 'active'`; verify `plan_id` is updated; verify period dates are unchanged.
- Test `create-subscription` 409: calling it twice for the same company returns a conflict.
- Test billing contact upsert: first call creates a record; second call updates it.
- Test `amount` and `price` are correctly represented in cents in all responses.
- Verify `features` is returned as a string (JSON-serialized array), not a parsed array.
- Test that a company cannot access another company's billing data using a different JWT.

---

# Open Questions / Missing Details

- **Stripe integration:** Stripe fields (`stripe_subscription_id`, `stripe_payment_id`) are present but Stripe webhook handling, payment initiation, and key management are not documented in this module.
- **`POST /billing/create-subscription` in production:** This endpoint is described as testing/seeding only. There is no documented access control beyond Bearer JWT. It should be gated or removed before production deployment.
- **Invoice generation:** `invoice_url` and `invoice_number` are stored in `payments` but the process for generating invoices is not documented.
- **Automated renewal/expiry:** There is no documented background job or scheduled task to automatically renew subscriptions or transition their status to `expired` when `current_period_end` passes.
- **Downgrade:** No downgrade endpoint exists. If a company wants to move to a lower-tier plan, the upgrade endpoint does not guard against this.
- **`billing_contacts.is_default` flag:** The `is_default` field is present but no logic is documented for how it is set or used, given there is only one billing contact per company.
- **PUT `/billing/contact` error responses:** No explicit error responses beyond 200 are documented for the contact upsert endpoint in the source. Error handling should be confirmed.
- **`features` field format:** The `features` column stores a JSON-encoded string. It is unclear whether the API deserializes this before returning it, or returns the raw string.

---

# Change Log / Notes

| Date       | Note                                                                   |
|------------|------------------------------------------------------------------------|
| 2026-03-31 | Initial standardized documentation pass from source notes              |
