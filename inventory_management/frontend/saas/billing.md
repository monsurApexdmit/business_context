# Frontend Module: SaaS Billing & Subscriptions

## Pages
- `/dashboard/billing` — Current subscription, plan details, payment history
- `/dashboard/billing/plans` — Plan comparison and upgrade/downgrade
- `/dashboard/billing/payment-methods` — Saved cards and payment methods
- `/dashboard/billing/invoices` — Invoice history with download links

## API Service
`lib/saasBillingApi.ts`

---

## GET /billing/plans
**Purpose:** Fetch all available subscription plans with features and pricing
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected from `localStorage.company_id`)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "plans": [
      {
        "id": 1,
        "name": "Starter",
        "description": "string",
        "price": 29.00,
        "currency": "USD",
        "billingPeriod": "monthly | yearly",
        "durationDays": 30,
        "isPopular": false,
        "maxUsers": 5,
        "maxBranches": 1,
        "maxProducts": 100,
        "features": [
          { "id": 1, "name": "string", "description": "string" }
        ],
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

**Frontend Impact:**
- Plan selection cards on upgrade/pricing page
- `isPopular` renders a "Most Popular" badge
- `maxUsers`, `maxBranches`, `maxProducts` shown in plan comparison table

---

## GET /billing/subscription/current
**Purpose:** Fetch the company's active subscription details
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
    "planId": 2,
    "planName": "Pro",
    "status": "active | cancelled | expired | pending",
    "currentPeriodStart": "2024-01-01T00:00:00Z",
    "currentPeriodEnd": "2024-02-01T00:00:00Z",
    "price": 79.00,
    "currency": "USD",
    "billingPeriod": "monthly | yearly",
    "nextBillingDate": "2024-02-01T00:00:00Z",
    "autoRenew": true,
    "licenseKey": "XXXX-XXXX-XXXX-XXXX",
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Billing dashboard shows current plan name, price, period
- `status: "expired"` shows renew prompt
- `autoRenew` toggle state sourced from this response
- `nextBillingDate` displayed in billing summary

---

## GET /billing/payments/history
**Purpose:** Paginated list of past payments/invoices
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)
- `page` (default: 1)
- `limit` (default: 10)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "payments": [
      {
        "id": 1,
        "subscriptionId": 1,
        "companyId": 1,
        "amount": 79.00,
        "currency": "USD",
        "status": "pending | paid | failed | cancelled",
        "paymentMethod": "card",
        "paymentDate": "2024-01-01T00:00:00Z",
        "invoiceNumber": "INV-0001",
        "invoiceUrl": "string",
        "transactionId": "string",
        "metadata": {},
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ],
    "total": 24,
    "page": 1,
    "limit": 10
  }
}
```

**Frontend Impact:**
- Payment history table with pagination
- Status badges (paid/failed/pending)
- Download invoice button links to `invoiceUrl` or calls `downloadInvoice`

---

## POST /billing/payments/process
**Purpose:** Process a new payment to create or renew a subscription
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "planId": 2,
  "planName": "Pro",
  "amount": 79.00,
  "currency": "USD",
  "billingPeriod": "monthly | yearly",
  "paymentMethodId": "pm_xxx",
  "stripeTokenId": "tok_xxx",
  "autoRenew": true,
  "useExistingPaymentMethod": false
}
```

**Note:** Either `paymentMethodId` (saved card), `stripeTokenId` (new card tokenized client-side), or `useExistingPaymentMethod: true` for the default saved method.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "paymentId": 1,
    "subscriptionId": 1,
    "status": "paid | pending",
    "licenseKey": "XXXX-XXXX-XXXX-XXXX",
    "subscriptionStartDate": "2024-01-01T00:00:00Z",
    "subscriptionEndDate": "2024-02-01T00:00:00Z",
    "invoiceUrl": "string",
    "nextBillingDate": "2024-02-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- On success: stores updated subscription info, shows success modal with license key
- On `status: "pending"`: shows "processing" state
- Subscription status in localStorage updated to reflect new active plan

---

## POST /billing/subscription/upgrade
**Purpose:** Upgrade current subscription to a higher-tier plan
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "newPlanId": 3,
  "prorationBillingCycle": "immediate | next_billing"
}
```

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "subscriptionId": 1,
    "planId": 3,
    "planName": "Enterprise",
    "price": 149.00,
    "prorationCredit": 25.50,
    "additionalCharge": 53.50,
    "status": "active | pending_upgrade",
    "createdAt": "2024-01-01T00:00:00Z"
  }
}
```

**Frontend Impact:**
- Upgrade confirmation modal shows `prorationCredit` and `additionalCharge` before confirming
- On success, subscription page refreshes with new plan details

---

## POST /billing/subscription/cancel
**Purpose:** Cancel the current active subscription
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "reason": "string",
  "feedback": "string"
}
```

**Note:** Both fields are optional. Backend may use for churn analytics.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "subscriptionId": 1,
    "status": "cancelled",
    "cancelledAt": "2024-01-01T00:00:00Z",
    "finalBillingDate": "2024-02-01T00:00:00Z",
    "refund": 0.00
  }
}
```

**Frontend Impact:**
- Cancellation confirmation modal with `finalBillingDate` shown as access end date
- Subscription page updates to show cancelled state with end date

---

## PATCH /billing/subscriptions/auto-renew
**Purpose:** Toggle automatic renewal on/off for the current subscription
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "autoRenew": true
}
```

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "subscriptionId": 1,
    "autoRenew": true
  }
}
```

**Frontend Impact:**
- Toggle switch on billing page reflects current state
- Warning shown when disabling auto-renew ("Your subscription will expire on X")

---

## POST /billing/subscription/renew
**Purpose:** Manually renew an expired or cancelled subscription
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "subscriptionId": 1,
  "paymentMethodId": "pm_xxx",
  "autoRenew": true
}
```

**Note:** `paymentMethodId` optional (uses default saved method if omitted). `autoRenew` optional.

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "subscriptionId": 1,
    "status": "active",
    "currentPeriodEnd": "2024-02-01T00:00:00Z",
    "nextBillingDate": "2024-02-01T00:00:00Z",
    "licenseKey": "XXXX-XXXX-XXXX-XXXX",
    "invoiceUrl": "string"
  }
}
```

**Frontend Impact:**
- Expired subscription banner replaced with active subscription view
- License key displayed/copied to clipboard

---

## POST /billing/payment-methods
**Purpose:** Save a new payment method (credit/debit card)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "stripeTokenId": "tok_xxx",
  "cardLastFour": "4242",
  "cardBrand": "visa",
  "cardExpiryMonth": 12,
  "cardExpiryYear": 2026,
  "billingName": "string",
  "billingEmail": "string",
  "billingAddress": "string",
  "billingCity": "string",
  "billingState": "string",
  "billingZip": "string",
  "billingCountry": "string"
}
```

**Note:** `billingName`, `billingEmail`, `billingAddress`, `billingCity`, `billingState`, `billingZip`, `billingCountry` are required. Card fields are optional (sourced from Stripe token).

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "paymentMethodId": "pm_xxx",
    "last4": "4242",
    "brand": "visa",
    "expiryMonth": 12,
    "expiryYear": 2026
  }
}
```

**Frontend Impact:**
- New card appears in saved payment methods list
- Card can be selected for future payments

---

## GET /billing/payment-methods
**Purpose:** Fetch all saved payment methods for the company
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "paymentMethods": [
      {
        "id": "pm_xxx",
        "last4": "4242",
        "brand": "visa",
        "expiryMonth": 12,
        "expiryYear": 2026,
        "isDefault": true
      }
    ]
  }
}
```

**Frontend Impact:**
- Payment method selector in checkout/upgrade flow
- Default card highlighted

---

## DELETE /billing/payment-methods/:id
**Purpose:** Remove a saved payment method
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — payment method ID string (e.g., `pm_xxx`)

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "Payment method deleted successfully"
}
```

**Frontend Impact:**
- Card removed from saved methods list
- If it was the default method, user prompted to set a new default

---

## GET /billing/invoices/:id/download
**Purpose:** Get download URL or PDF for a specific invoice
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Path Params:**
- `id` — invoice ID (numeric)

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "url": "https://cdn.example.com/invoices/INV-0001.pdf",
  "filename": "INV-0001.pdf"
}
```

**Note:** Response does NOT use the standard `{ message, data }` envelope — it returns `{ url, filename }` directly.

**Frontend Impact:**
- "Download" button in payment history opens `url` in new tab or triggers browser download

---

## GET /billing/trial/info
**Purpose:** Fetch trial status and remaining days
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Expected Response 200:**
```json
{
  "message": "string",
  "data": {
    "isTrialActive": true,
    "trialStartDate": "2024-01-01T00:00:00Z",
    "trialEndDate": "2024-01-14T00:00:00Z",
    "daysRemaining": 7,
    "daysUsed": 7,
    "trialLicenseKey": "TRIAL-XXXX-XXXX"
  }
}
```

**Frontend Impact:**
- Trial countdown banner shown when `isTrialActive: true`
- `daysRemaining` shown prominently to drive conversion
- After trial expires (`daysRemaining <= 0`), upgrade prompt shown

---

## POST /billing/trial/extend
**Purpose:** Extend the trial period (admin/superadmin operation)
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Query Params:**
- `company_id` (auto-injected)

**Request Body:**
```json
{
  "days": 7
}
```

**Expected Response 200:**
```json
{
  "message": "Trial extended successfully"
}
```

**Frontend Impact:**
- Admin panel only — not exposed in regular company UI
- After success, trial info is re-fetched to show updated end date
