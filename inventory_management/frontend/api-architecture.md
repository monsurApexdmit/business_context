# Frontend API Architecture

## Tech Stack
- **Framework:** Next.js (App Router) with React 19, TypeScript
- **HTTP Client:** Axios (one instance per API service module)
- **Proxy:** All requests go through `/api/proxy/[...path]` (Next.js route) → backend at `https://localhost:8004`

## Base URL
All axios instances use `baseURL: '/api/proxy'`.
The proxy forwards to `BACKEND_URL` from `.env.local`.

## Auth Token
- Stored in `localStorage` key: `token`
- Sent on every request as: `Authorization: Bearer <token>`
- Also stored in cookie `token` (for Next.js middleware route protection)
- On `401` response: token + company_id removed from localStorage, redirect to `/auth/login`

## Multi-Tenancy
- `company_id` stored in `localStorage` key: `company_id`
- Automatically appended to **every** request as query param: `?company_id=<id>`
- For FormData requests: appended as a FormData field
- Source: `lib/utils/apiInterceptor.ts` → `getCompanyId()`

## Additional localStorage Keys
| Key | Type | Purpose |
|-----|------|---------|
| `token` | string | JWT Bearer token |
| `company_id` | string (int) | Active company ID |
| `user_role` | string | `owner \| admin \| manager \| staff` |
| `trial_days` | string (int) | Days remaining in trial |

## Request Interceptor (all modules)
```
1. Read token from localStorage → set Authorization header
2. Read company_id from localStorage → append to params
```

## Response Interceptor (all modules)
```
401 → clear localStorage token + company_id + user_role + trial_days
     → clear cookie token
     → redirect to /auth/login
```

## Content-Type Rules
| Scenario | Content-Type |
|----------|-------------|
| JSON requests | `application/json` |
| Product create/update | `multipart/form-data` (auto via FormData) |
| Coupon with image | `multipart/form-data` (Content-Type header deleted so browser sets boundary) |
| Settings logo/banner upload | `multipart/form-data` |
| All other requests | `application/json` |

## Route Protection
Next.js `middleware.ts` checks `token` cookie on all `/dashboard/*` routes.
If missing → redirect to `/auth/login`.

## API Service Files
| File | Covers |
|------|--------|
| `lib/saasAuthApi.ts` | signup, login, logout, /auth/me, password |
| `lib/saasCompanyApi.ts` | company profile, settings, team management |
| `lib/saasBillingApi.ts` | plans, subscriptions, payments |
| `lib/productApi.ts` | products CRUD (multipart) |
| `lib/categoryApi.ts` | categories CRUD |
| `lib/attributeApi.ts` | attributes CRUD |
| `lib/inventoryApi.ts` | inventory read-only |
| `lib/locationApi.ts` | locations CRUD |
| `lib/transferApi.ts` | stock transfers |
| `lib/sellsApi.ts` | orders (sells) CRUD |
| `lib/couponApi.ts` | coupons CRUD + validate |
| `lib/shipmentsApi.ts` | shipments CRUD |
| `lib/customerApi.ts` | customers CRUD |
| `lib/customerReturnsApi.ts` | customer returns CRUD |
| `lib/vendorApi.ts` | vendors CRUD |
| `lib/vendorReturnsApi.ts` | vendor returns CRUD |
| `lib/staffApi.ts` | staff + staff roles + salary payments |
| `lib/settingsApi.ts` | store settings (all sections) |
