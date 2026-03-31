# Module: SaaS Billing

## Business Purpose
Manages subscription plans, company subscriptions, payment history, and billing contacts. Handles subscription lifecycle: viewing, renewing, cancelling, and upgrading. Also manages billing contact information for invoicing.

---

## Database Tables

### Table: `subscription_plans`
| Column         | Type          | Notes                                         |
|----------------|---------------|-----------------------------------------------|
| id             | uint (PK, AI) |                                               |
| name           | string        | json: name                                    |
| description    | string        | json: description                             |
| price          | int64         | price in cents; json: price                   |
| billing_period | string        | 'monthly', 'yearly'; json: billingPeriod      |
| max_users      | int           | json: maxUsers                                |
| max_products   | int           | json: maxProducts                             |
| max_branches   | int           | json: maxBranches                             |
| features       | string        | JSON string of features array; json: features |
| is_active      | bool          | json: isActive                                |
| is_featured    | bool          | json: isFeatured                              |
| created_at     | timestamp     | json: createdAt                               |
| updated_at     | timestamp     |                                               |

### Table: `subscriptions`
| Column                 | Type          | Notes                                                  |
|------------------------|---------------|--------------------------------------------------------|
| id                     | uint (PK, AI) |                                                        |
| company_id             | uint (FK)     | FK → companies.id                                      |
| plan_id                | uint (FK)     | FK → subscription_plans.id                             |
| status                 | string        | 'active','expired','cancelled','pending'; json: status |
| current_period_start   | timestamp     | json: currentPeriodStart                               |
| current_period_end     | timestamp     | json: currentPeriodEnd                                 |
| next_billing_date      | timestamp     | json: nextBillingDate                                  |
| auto_renew             | bool          | json: autoRenew                                        |
| stripe_subscription_id | string        | json: stripeSubscriptionId                             |
| cancelled_at           | *timestamp    | nullable; json: cancelledAt                            |
| created_at             | timestamp     | json: createdAt                                        |
| updated_at             | timestamp     |                                                        |

### Table: `payments`
| Column            | Type          | Notes                                              |
|-------------------|---------------|----------------------------------------------------|
| id                | uint (PK, AI) |                                                    |
| subscription_id   | uint (FK)     | FK → subscriptions.id                              |
| company_id        | uint (FK)     | FK → companies.id                                  |
| amount            | int64         | in cents; json: amount                             |
| status            | string        | 'paid','pending','failed'; json: status            |
| payment_method    | string        | json: paymentMethod                                |
| payment_date      | timestamp     | json: paymentDate; ordered DESC by default         |
| invoice_number    | string        | json: invoiceNumber                                |
| invoice_url       | string        | json: invoiceUrl                                   |
| stripe_payment_id | string        | json: stripePaymentId                              |
| description       | string        | json: description                                  |
| created_at        | timestamp     | json: createdAt                                    |
| updated_at        | timestamp     |                                                    |

### Table: `billing_contacts`
| Column     | Type          | Notes                                          |
|------------|---------------|------------------------------------------------|
| id         | uint (PK, AI) |                                                |
| company_id | uint          | unique index; json: companyId                  |
| email      | string        | json: email                                    |
| phone      | string        | json: phone                                    |
| address    | string        | json: address                                  |
| city       | string        | json: city                                     |
| state      | string        | json: state                                    |
| zip_code   | string        | json: zipCode                                  |
| country    | string        | json: country                                  |
| tax_id     | string        | json: taxId                                    |
| tax_id_type| string        | 'vat','ein','gst','other'; json: taxIdType     |
| is_default | bool          | json: isDefault                                |
| created_at | timestamp     | json: createdAt                                |
| updated_at | timestamp     | json: updatedAt                                |

---

## Relationships
- Subscription → SubscriptionPlan (BelongsTo via plan_id; preloaded on GetCurrentSubscription)
- Subscription ← Payments (HasMany)
- Company → Subscription (HasOne)
- Company → BillingContact (HasOne)

---

## Business Logic
- **GetPlans**: returns only active plans (`is_active = true`).
- **GetCurrentSubscription**: fetches latest subscription by `created_at DESC`; returns it regardless of status.
- **GetPaymentHistory**: all payments ordered by `payment_date DESC`.
- **RenewSubscription**: extends period from `current_period_end` by 1 month or 1 year based on plan's `billing_period`.
- **CancelSubscription**: sets `status = "cancelled"`, records `cancelled_at`.
- **UpgradeSubscription**: requires an `active` subscription; updates `plan_id`.
- **CreateSubscriptionForCompany**: testing/seeding endpoint — creates a 10-day subscription at plan_id 1 (trial) if no subscription exists.
- **UpdateBillingContact**: upsert pattern — finds existing or creates new record.

---

## Endpoints

### GET /billing/plans
**Auth:** Bearer JWT required
**Purpose:** List all active subscription plans.

**Response 200:**
```json
{
  "message": "Plans retrieved",
  "data": [
    {
      "id": 1,
      "name": "Trial",
      "description": "10-day free trial",
      "price": 0,
      "billingPeriod": "monthly",
      "maxUsers": 5,
      "maxProducts": 100,
      "maxBranches": 1,
      "features": "[\"Feature 1\",\"Feature 2\"]",
      "isActive": true,
      "isFeatured": false,
      "createdAt": "..."
    }
  ]
}
```

**Error:** `500` — `{"message": "Failed to fetch plans"}`

---

### GET /billing/subscription
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Subscription retrieved",
  "data": {
    "id": 1,
    "planId": 1,
    "planName": "Trial",
    "price": 0,
    "billingPeriod": "monthly",
    "status": "active",
    "currentPeriodStart": "2024-01-01T00:00:00Z",
    "currentPeriodEnd": "2024-01-11T00:00:00Z",
    "nextBillingDate": "2024-01-11T00:00:00Z",
    "autoRenew": true,
    "createdAt": "..."
  }
}
```

**Error Responses:**
- `401` — `{"message": "Unauthorized - company ID not found in token"}`
- `404` — `{"message": "No subscription found for this company"}`

---

### GET /billing/payments
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Payment history retrieved",
  "data": {
    "payments": [
      {
        "id": 1,
        "paymentDate": "2024-01-01T00:00:00Z",
        "createdAt": "...",
        "invoiceNumber": "INV-2024-001",
        "amount": 2999,
        "status": "paid",
        "invoiceUrl": ""
      }
    ]
  }
}
```

**Error Responses:**
- `401` — `{"message": "Unauthorized - company ID not found in token"}`
- `500` — `{"message": "Failed to fetch payment history"}`

---

### POST /billing/renew
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "subscriptionId": 1,
  "autoRenew": true
}
```

**Validation:**
- `subscriptionId` — required

**Response 200:**
```json
{
  "message": "Subscription renewed successfully",
  "data": {
    "currentPeriodEnd": "2024-02-11T00:00:00Z",
    "nextBillingDate": "2024-02-11T00:00:00Z",
    "status": "active"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request"}`
- `401` — `{"message": "Unauthorized - company ID not found in token"}`
- `404` — `{"message": "Subscription not found"}`
- `500` — `{"message": "Failed to renew subscription"}`

---

### POST /billing/cancel
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "subscriptionId": 1
}
```

**Response 200:**
```json
{
  "message": "Subscription cancelled successfully",
  "data": {
    "status": "cancelled",
    "cancelledAt": "2024-01-15T00:00:00Z"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request"}`
- `401` — `{"message": "Unauthorized - company ID not found in token"}`
- `404` — `{"message": "Subscription not found"}`
- `500` — `{"message": "Failed to cancel subscription"}`

---

### POST /billing/upgrade
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{
  "newPlanId": 2
}
```

**Validation:**
- `newPlanId` — required; must reference an existing plan

**Response 200:**
```json
{
  "message": "Subscription upgraded successfully",
  "data": {
    "newPlanId": 2,
    "newPlanName": "Professional"
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request"}`
- `401` — `{"message": "Unauthorized - company ID not found in token"}`
- `404` — `{"message": "No active subscription found"}` / `{"message": "Plan not found"}`
- `500` — `{"message": "Failed to upgrade subscription"}`

---

### POST /billing/create-subscription
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Testing/seeding endpoint — creates a subscription for a company.

**Request Body:**
```json
{
  "companyId": 1,
  "planId": 1
}
```

**Response 201:**
```json
{
  "message": "Subscription created successfully",
  "data": {
    "id": 1,
    "companyId": 1,
    "planId": 1,
    "status": "active",
    "currentPeriodStart": "...",
    "currentPeriodEnd": "...",
    "nextBillingDate": "...",
    "autoRenew": true
  }
}
```

**Error Responses:**
- `400` — `{"message": "Invalid request", "error": "..."}`
- `404` — `{"message": "Company not found"}` / `{"message": "Plan not found"}`
- `409` — `{"message": "Subscription already exists for this company"}`
- `500` — `{"message": "Failed to create subscription"}`

---

### GET /billing/contact
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Billing contact retrieved",
  "data": {
    "id": 1,
    "companyId": 1,
    "email": "billing@acme.com",
    "phone": "+1234567890",
    "address": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "taxId": "US123456789",
    "taxIdType": "ein",
    "isDefault": true,
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**Error Responses:**
- `400` — invalid company ID
- `404` — `{"message": "Billing contact not found"}`

---

### PUT /billing/contact
**Auth:** Bearer JWT required
**Content-Type:** `application/json`
**Purpose:** Upsert billing contact.

**Request Body (BillingContactRequest):**
```json
{
  "email": "billing@acme.com",
  "phone": "+1234567890",
  "address": "123 Main St",
  "city": "New York",
  "state": "NY",
  "zipCode": "10001",
  "country": "US",
  "taxId": "US123456789",
  "taxIdType": "ein"
}
```

**Response 200:**
```json
{
  "message": "Billing contact updated",
  "data": { /* BillingContactDTO */ }
}
```

---

## Dependencies
- **saas-team** — max_users from subscription plan enforces team size
- **saas-company** — company status affects subscription state
