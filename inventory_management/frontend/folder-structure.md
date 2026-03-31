# Frontend Folder Structure (Mirrored to Backend)

This structure maps the frontend to match the backend module layout,
making it easy to implement and maintain alongside the backend.

---

## Proposed Next.js App Router Structure

```
src/
├── app/
│   ├── api/
│   │   └── proxy/
│   │       └── [...path]/
│   │           └── route.ts                  # Proxy → backend https://localhost:8004
│   │
│   ├── (auth)/
│   │   └── auth/
│   │       ├── login/page.tsx
│   │       ├── signup/page.tsx
│   │       └── reset-password/page.tsx
│   │
│   └── dashboard/
│       ├── layout.tsx                        # Sidebar + auth guard
│       │
│       ├── (saas)/
│       │   ├── company/page.tsx              # ← backend: saas/saas-company
│       │   ├── team/page.tsx                 # ← backend: saas/saas-team
│       │   └── billing/
│       │       ├── page.tsx                  # ← backend: saas/saas-billing
│       │       └── blocked-access/page.tsx
│       │
│       ├── (catalog)/
│       │   ├── products/
│       │   │   ├── page.tsx                  # ← backend: products/
│       │   │   ├── new/page.tsx
│       │   │   └── [id]/
│       │   │       ├── page.tsx
│       │   │       └── edit/page.tsx
│       │   ├── categories/
│       │   │   ├── page.tsx                  # ← backend: categories/
│       │   │   └── [id]/page.tsx
│       │   ├── attributes/
│       │   │   ├── page.tsx                  # ← backend: attributes/
│       │   │   └── [id]/page.tsx
│       │   └── inventory/
│       │       └── page.tsx                  # ← backend: inventory/
│       │
│       ├── (sales)/
│       │   ├── orders/
│       │   │   ├── page.tsx                  # ← backend: orders/
│       │   │   ├── new/page.tsx
│       │   │   └── [id]/page.tsx
│       │   ├── coupons/
│       │   │   ├── page.tsx                  # ← backend: coupons/
│       │   │   └── [id]/page.tsx
│       │   ├── shipping/
│       │   │   ├── page.tsx                  # ← backend: shipping/
│       │   │   └── [id]/page.tsx
│       │   └── customer-returns/
│       │       ├── page.tsx                  # ← backend: customer-returns/
│       │       └── [id]/page.tsx
│       │
│       ├── (procurement)/
│       │   ├── vendors/
│       │   │   ├── page.tsx                  # ← backend: vendors/
│       │   │   ├── new/page.tsx
│       │   │   └── [id]/page.tsx
│       │   └── vendor-returns/
│       │       ├── page.tsx                  # ← backend: vendor-returns/
│       │       └── [id]/page.tsx
│       │
│       ├── (hr)/
│       │   ├── staff/
│       │   │   ├── page.tsx                  # ← backend: staff/
│       │   │   ├── new/page.tsx
│       │   │   └── [id]/page.tsx
│       │   ├── staff-roles/
│       │   │   ├── page.tsx                  # ← backend: staff-roles/
│       │   │   └── [id]/page.tsx
│       │   └── salary-payments/
│       │       ├── page.tsx                  # ← backend: salary-payments/
│       │       └── [id]/page.tsx
│       │
│       ├── (operations)/
│       │   ├── locations/
│       │   │   ├── page.tsx                  # ← backend: locations/
│       │   │   └── [id]/page.tsx
│       │   ├── stock-transfers/
│       │   │   ├── page.tsx                  # ← backend: stock-transfers/
│       │   │   ├── new/page.tsx
│       │   │   └── [id]/page.tsx
│       │   └── settings/
│       │       └── page.tsx                  # ← backend: settings/
│       │
│       └── customers/
│           ├── page.tsx                      # ← backend: customers/
│           ├── new/page.tsx
│           └── [id]/page.tsx
│
├── lib/
│   ├── axios.ts                              # Shared axios factory (base URL + interceptors)
│   │
│   ├── saas/
│   │   ├── saasAuthApi.ts                    # ← backend: auth/
│   │   ├── saasCompanyApi.ts                 # ← backend: saas/saas-company
│   │   ├── saasBillingApi.ts                 # ← backend: saas/saas-billing
│   │   └── saasTeamApi.ts                    # ← backend: saas/saas-team
│   │
│   ├── catalog/
│   │   ├── productApi.ts                     # ← backend: products/
│   │   ├── categoryApi.ts                    # ← backend: categories/
│   │   ├── attributeApi.ts                   # ← backend: attributes/
│   │   └── inventoryApi.ts                   # ← backend: inventory/
│   │
│   ├── sales/
│   │   ├── sellsApi.ts                       # ← backend: orders/
│   │   ├── couponApi.ts                      # ← backend: coupons/
│   │   ├── shipmentsApi.ts                   # ← backend: shipping/
│   │   └── customerReturnsApi.ts             # ← backend: customer-returns/
│   │
│   ├── procurement/
│   │   ├── vendorApi.ts                      # ← backend: vendors/
│   │   └── vendorReturnsApi.ts               # ← backend: vendor-returns/
│   │
│   ├── hr/
│   │   ├── staffApi.ts                       # ← backend: staff/
│   │   ├── staffRolesApi.ts                  # ← backend: staff-roles/
│   │   └── salaryPaymentsApi.ts              # ← backend: salary-payments/
│   │
│   ├── operations/
│   │   ├── locationApi.ts                    # ← backend: locations/
│   │   ├── transferApi.ts                    # ← backend: stock-transfers/
│   │   └── settingsApi.ts                    # ← backend: settings/
│   │
│   ├── customers/
│   │   └── customerApi.ts                    # ← backend: customers/
│   │
│   └── utils/
│       ├── apiInterceptor.ts                 # Injects token + company_id on every request
│       └── auth.ts                           # localStorage helpers (getToken, getCompanyId, etc.)
│
├── components/
│   ├── ui/                                   # Shared UI primitives (Button, Input, Modal, Table...)
│   │
│   ├── saas/
│   │   ├── LoginForm.tsx
│   │   ├── SignupForm.tsx
│   │   └── BillingCard.tsx
│   │
│   ├── catalog/
│   │   ├── ProductForm.tsx
│   │   ├── ProductTable.tsx
│   │   ├── CategoryForm.tsx
│   │   ├── AttributeForm.tsx
│   │   ├── InventoryTable.tsx                # ← inventory/  (read-only, per-location breakdown)
│   │   └── InventoryFilters.tsx
│   │
│   ├── sales/
│   │   ├── OrderForm.tsx
│   │   ├── OrderTable.tsx
│   │   ├── CouponForm.tsx
│   │   ├── ShipmentTable.tsx
│   │   └── CustomerReturnForm.tsx
│   │
│   ├── procurement/
│   │   ├── VendorForm.tsx
│   │   └── VendorReturnForm.tsx
│   │
│   ├── hr/
│   │   ├── StaffForm.tsx
│   │   ├── StaffRoleForm.tsx
│   │   └── SalaryPaymentTable.tsx
│   │
│   ├── operations/
│   │   ├── LocationForm.tsx
│   │   ├── StockTransferForm.tsx
│   │   └── SettingsForm.tsx
│   │
│   └── customers/
│       ├── CustomerForm.tsx
│       └── CustomerTable.tsx
│
├── types/
│   ├── catalog.ts                            # Product, Variant, Category, Attribute, InventoryRow
│   ├── sales.ts                              # Order, Coupon, Shipment, CustomerReturn
│   ├── procurement.ts                        # Vendor, VendorReturn
│   ├── hr.ts                                 # Staff, StaffRole, SalaryPayment
│   ├── operations.ts                         # Location, StockTransfer, Settings
│   ├── customers.ts                          # Customer
│   └── saas.ts                               # Company, Team, Billing, AuthUser
│
└── middleware.ts                             # Protects /dashboard/* → redirect to /auth/login if no token
```

---

## Backend ↔ Frontend Module Map

| Backend Module         | Frontend API File (`lib/`)               | Frontend Page (`app/dashboard/`)          |
|------------------------|------------------------------------------|-------------------------------------------|
| `auth/`                | `saas/saasAuthApi.ts`                    | `(auth)/auth/login`, `signup`             |
| `saas/saas-company`    | `saas/saasCompanyApi.ts`                 | `(saas)/company/`                         |
| `saas/saas-team`       | `saas/saasTeamApi.ts`                    | `(saas)/team/`                            |
| `saas/saas-billing`    | `saas/saasBillingApi.ts`                 | `(saas)/billing/`                         |
| `products/`            | `catalog/productApi.ts`                  | `(catalog)/products/`                     |
| `categories/`          | `catalog/categoryApi.ts`                 | `(catalog)/categories/`                   |
| `attributes/`          | `catalog/attributeApi.ts`                | `(catalog)/attributes/`                   |
| `inventory/`           | `catalog/inventoryApi.ts`                | `(catalog)/inventory/`                    |
| `orders/`              | `sales/sellsApi.ts`                      | `(sales)/orders/`                         |
| `coupons/`             | `sales/couponApi.ts`                     | `(sales)/coupons/`                        |
| `shipping/`            | `sales/shipmentsApi.ts`                  | `(sales)/shipping/`                       |
| `customer-returns/`    | `sales/customerReturnsApi.ts`            | `(sales)/customer-returns/`               |
| `vendors/`             | `procurement/vendorApi.ts`               | `(procurement)/vendors/`                  |
| `vendor-returns/`      | `procurement/vendorReturnsApi.ts`        | `(procurement)/vendor-returns/`           |
| `staff/`               | `hr/staffApi.ts`                         | `(hr)/staff/`                             |
| `staff-roles/`         | `hr/staffRolesApi.ts`                    | `(hr)/staff-roles/`                       |
| `salary-payments/`     | `hr/salaryPaymentsApi.ts`                | `(hr)/salary-payments/`                   |
| `locations/`           | `operations/locationApi.ts`              | `(operations)/locations/`                 |
| `stock-transfers/`     | `operations/transferApi.ts`              | `(operations)/stock-transfers/`           |
| `settings/`            | `operations/settingsApi.ts`              | `(operations)/settings/`                  |
| `customers/`           | `customers/customerApi.ts`               | `customers/`                              |

---

## Key Design Rules

1. **One API file per backend module** — `lib/<group>/<module>Api.ts` mirrors `backend/<module>/`
2. **Same group names** — `catalog/`, `sales/`, `procurement/`, `hr/`, `operations/` used in `lib/`, `components/`, `types/`, and `app/dashboard/`
3. **Shared interceptor** — `lib/utils/apiInterceptor.ts` injects `Authorization` + `company_id` into every axios instance
4. **Proxy route** — All requests go to `/api/proxy/[...path]` → backend, never directly from browser
5. **Type files** — Each group has one `types/<group>.ts` with all request/response types for that group
