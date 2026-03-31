# Module Overview

**Module Name:** POS (Point of Sale)
**Module Path:** `app/dashboard/pos/`
**Route:** `/dashboard/pos`
**Navigation:** Sidebar → "POS" (CreditCard icon)

The POS module is a fully-functional in-store point-of-sale interface. It allows staff to browse the product catalog, build a cart, apply discounts and coupons, collect payment, and generate a receipt — all in a single-page session. Orders created through POS are submitted to the same backend Sells API used by the regular orders module.

---

# Business Context

POS exists as an alternative to the Orders module for physical in-store or counter sales. It is optimised for speed:
- Staff do not need to navigate multiple pages
- The entire sale — product selection, cart, discount, payment — happens on one screen
- Walk-in customers can be served without creating a customer record
- Receipts can be printed immediately on a thermal printer (80mm)

Orders created through POS appear in the Orders list with `status: "Pending"` and are indistinguishable from orders created elsewhere.

---

# Functional Scope

## Included

- Product browsing with search and category filtering
- Warehouse/location selector to show correct stock levels
- Product variant selection (when a product has variants)
- Real-time available stock tracking (total stock minus items already in cart)
- Cart management (add, increase/decrease quantity, remove)
- Customer selection (from customer list) or walk-in sale (no customer)
- Manual discount — fixed amount or percentage
- Coupon code discount — applies from the active coupon list
- Tax calculation (10% fixed on subtotal)
- Checkout modal — payment method selection (Cash / Card / Online)
- Card payment form (card number, expiry, CVV) shown when Card selected
- Order submission to backend Sells API
- Success modal with invoice number and order summary
- Thermal receipt printing (80mm, browser print dialog)
- Cart reset with confirmation dialog

## Excluded

- Shipping address collection (POS is in-store only)
- Refunds or returns (handled by the Customer Returns module)
- Stock management or inventory adjustments
- Creating new customers from within POS (must exist already)
- Saving a cart / hold orders (no draft/hold functionality)
- Barcode scanner integration (search is text-only)
- Split payments (only one payment method per transaction)

---

# Architecture Notes

- **Framework:** Next.js (App Router) with React 19 and TypeScript
- **Styling:** Tailwind CSS 4 with shadcn/ui (Radix UI primitives)
- **State management:** React Context API for global data (products, customers, categories, warehouses) + local `useState` for all POS session state (cart, discounts, modals)
- **HTTP:** Axios through a Next.js API proxy at `/api/proxy/[...path]` — avoids TLS issues with self-signed backend certificates
- **Auth:** Bearer token auto-injected by API interceptor (`/lib/utils/apiInterceptor.ts`); `company_id` also auto-injected for multi-tenancy
- **No routing between sub-pages** — POS is entirely one page; secondary actions use modal dialogs
- All cart state is ephemeral (in-memory); it does not persist across page refreshes

---

# Page Structure

## Layout

The POS page uses a **two-column layout**:

```
┌────────────────────────────────┬───────────────────────────┐
│  LEFT PANEL (60%)              │  RIGHT PANEL (40%)        │
│  ─ Search bar                  │  ─ Customer selector      │
│  ─ Warehouse selector          │  ─ Cart items list        │
│  ─ Category tabs               │  ─ Cart actions           │
│  ─ Product grid                │    (Discount / Coupon /   │
│    (2–5 columns responsive)    │     Reset)                │
│                                │  ─ Price summary          │
│                                │  ─ Pay Now button         │
└────────────────────────────────┴───────────────────────────┘
```

## Modals

| Modal | Trigger | Purpose |
|-------|---------|---------|
| Variant Selection | Clicking a product with variants | Choose size/color before adding to cart |
| Discount Modal | "Discount" button | Apply fixed or percentage manual discount |
| Coupon Modal | "Coupon" button | Select and apply an active coupon |
| Checkout Modal | "Pay Now" button | Select payment method; optional card form |
| Success Modal | After order is created | Show invoice number and print receipt |
| Reset Confirmation | "Reset" button | Confirm clearing the entire cart |

---

# Components

## File Map

| File | Path | Responsibility |
|------|------|---------------|
| `PosPage` | `app/dashboard/pos/page.tsx` | Root component; orchestrates all state and child components |
| `PosProductCard` | `components/pos/pos-product-card.tsx` | Displays one product in the grid; shows image, name, price, add-to-cart button |
| `PosCartItem` | `components/pos/pos-cart-item.tsx` | Renders one cart row; quantity +/- controls, remove button |
| `CustomerCombobox` | `components/pos/customer-combobox.tsx` | Searchable customer dropdown; includes Walk-in option |
| `DiscountModal` | `components/pos/discount-modal.tsx` | Fixed or percentage discount input with live preview |
| `CheckoutModal` | `components/pos/checkout-modal.tsx` | Payment method selection; card form when Card selected |
| `SuccessModal` | `components/pos/success-modal.tsx` | Order confirmation; invoice number; print receipt button |

## Component Props

### `PosProductCard`
| Prop | Type | Description |
|------|------|-------------|
| `product` | `Product` | Full product object |
| `onAddToCart` | `(product, variant?) => void` | Called when user clicks the product |
| `availableStock` | `number` | Stock minus items already in cart |

### `PosCartItem`
| Prop | Type | Description |
|------|------|-------------|
| `item` | `CartItem` | Cart item data |
| `onIncrease` | `() => void` | Increment quantity |
| `onDecrease` | `() => void` | Decrement quantity (removes if qty reaches 0) |
| `onRemove` | `() => void` | Remove item from cart entirely |
| `maxQuantity` | `number` | Available stock limit |

### `CustomerCombobox`
| Prop | Type | Description |
|------|------|-------------|
| `customers` | `Customer[]` | Full customer list from context |
| `selectedId` | `string` | Currently selected customer ID |
| `onSelect` | `(id, name) => void` | Called when customer is chosen |

### `DiscountModal`
| Prop | Type | Description |
|------|------|-------------|
| `isOpen` | `boolean` | Controls visibility |
| `onClose` | `() => void` | Close handler |
| `subtotal` | `number` | Current cart subtotal for live preview |
| `onApply` | `(amount, type) => void` | Called with discount value and type |

### `CheckoutModal`
| Prop | Type | Description |
|------|------|-------------|
| `isOpen` | `boolean` | Controls visibility |
| `onClose` | `() => void` | Close handler |
| `total` | `number` | Final amount to charge |
| `onConfirm` | `(method, cardDetails?) => void` | Called when payment confirmed |
| `isSubmitting` | `boolean` | Shows loading state on confirm button |

### `SuccessModal`
| Prop | Type | Description |
|------|------|-------------|
| `isOpen` | `boolean` | Controls visibility |
| `invoiceNo` | `string` | Generated invoice number |
| `orderSummary` | `OrderSummary` | Items, totals, customer name |
| `onClose` | `() => void` | Resets session and closes modal |

---

# State Management

## Contexts Consumed

| Context | File | Data Used in POS |
|---------|------|-----------------|
| `ProductContext` | `contexts/product-context.tsx` | Product list, inventory by warehouse |
| `CustomerContext` | `contexts/customer-context.tsx` | Customer list for combobox |
| `CategoryContext` | `contexts/category-context.tsx` | Flat category list for tabs |
| `WarehouseContext` | `contexts/warehouse-context.tsx` | Warehouse list, default warehouse |

## Local State (PosPage)

| State Variable | Type | Purpose |
|----------------|------|---------|
| `products` | `Product[]` | Filtered product list for current view |
| `searchQuery` | `string` | Text search filter |
| `selectedCategory` | `string` | Active category tab (`"all"` or category ID) |
| `selectedWarehouseId` | `string` | Active warehouse for stock lookup |
| `cart` | `CartItem[]` | Current cart items |
| `selectedCustomerId` | `string` | Selected customer ID (empty = walk-in) |
| `selectedCustomerName` | `string` | Display name (default: `"Walk-in Customer"`) |
| `discount` | `number` | Manual discount amount (in currency) |
| `appliedCoupon` | `CouponResponse \| null` | Currently applied coupon object |
| `couponDiscount` | `number` | Calculated discount from coupon |
| `availableCoupons` | `CouponResponse[]` | Loaded coupon list for modal |
| `isCheckoutOpen` | `boolean` | Checkout modal visibility |
| `isDiscountModalOpen` | `boolean` | Discount modal visibility |
| `isCouponModalOpen` | `boolean` | Coupon modal visibility |
| `isResetDialogOpen` | `boolean` | Reset confirmation dialog visibility |
| `isSuccessModalOpen` | `boolean` | Success modal visibility |
| `invoiceNo` | `string` | Invoice number returned after order creation |
| `isSubmitting` | `boolean` | Prevents double-submission during API call |

---

# Data Types / Interfaces

### `CartItem`
```typescript
interface CartItem {
  id: string           // "{productId}" or "{productId}-{variantId}"
  productId: number
  variantId?: number
  inventoryId?: number // Used in order items for stock deduction
  name: string
  price: number        // Sale price at the time of adding to cart
  image: string
  quantity: number
  variantName?: string
}
```

### `Product` (from ProductContext)
```typescript
interface Product {
  id: string
  name: string
  description: string
  category: string
  categoryId?: string
  price: number
  salePrice: number
  stock: number
  status: "Selling" | "Out of Stock" | "Discontinued"
  published: boolean
  image: string
  sku: string
  barcode: string
  variants?: Variant[]
  inventory?: {
    inventoryId?: number
    warehouseId: string
    quantity: number
  }[]
}
```

### `Variant` (nested in Product)
```typescript
interface Variant {
  id: string
  name: string
  attributes: { [key: string]: string }
  price: number
  salePrice: number
  stock: number
  sku: string
  barcode?: string
  inventory?: {
    inventoryId?: number
    warehouseId: string
    quantity: number
  }[]
}
```

### `CouponResponse` (from `lib/couponApi.ts`)
```typescript
interface CouponResponse {
  id: number
  campaign_name: string
  code: string
  discount: number
  type: 'percentage' | 'fixed'
  status: boolean
  start_date: string
  end_date: string
  usage_limit: number | null
  min_order_amount: number
  max_discount: number | null
}
```

### `CreateSellData` (sent to backend on checkout)
```typescript
interface CreateSellData {
  customerId?: number
  customerName: string
  customerEmail?: string
  customerPhone?: string
  method: 'Cash' | 'Card' | 'Online'
  amount: number
  discount?: number
  couponId?: number
  couponCode?: string
  shippingCost?: number
  status: 'Pending'
  items: {
    productId: number
    variantId?: number
    inventory_id?: number
    productName: string
    quantity: number
    price: number
  }[]
}
```

---

# API Integration

## Endpoints Called

| API Call | Method | Endpoint | When Called |
|----------|--------|----------|-------------|
| Load products | GET | `/products/` | On mount; on warehouse or category change |
| Load customers | GET | `/customers/` | Via CustomerContext on app load |
| Load coupons | GET | `/coupons/` | When coupon modal is opened |
| Create order | POST | `/sells/` | On checkout confirmation |

## Load Products — Query Parameters

| Param | Value | Notes |
|-------|-------|-------|
| `limit` | `100` | Loads full catalog for client-side filtering |
| `location_id` | `selectedWarehouseId` | Filters inventory to selected warehouse |
| `category_id` | `selectedCategory` | Optional; filters by category |

## Create Order — Request Body

```json
{
  "customerName": "Walk-in Customer",
  "customerId": 5,
  "method": "Cash",
  "amount": 109.89,
  "discount": 10.00,
  "couponId": 3,
  "couponCode": "SUMMER20",
  "shippingCost": 0,
  "status": "Pending",
  "items": [
    {
      "productId": 12,
      "variantId": 7,
      "inventory_id": 4,
      "productName": "T-Shirt (Small / Red)",
      "quantity": 2,
      "price": 29.99
    }
  ]
}
```

## Create Order — Success Response

```json
{
  "message": "Sell created successfully",
  "data": {
    "id": 101,
    "invoiceNo": "INV-1700000000",
    "customerName": "Walk-in Customer",
    "amount": 109.89,
    "status": "Pending",
    "items": [ ... ]
  }
}
```

---

# Business Logic Details

## Price Calculations

```
subtotal      = SUM(item.price × item.quantity)
tax           = subtotal × 0.10  (10% fixed)
totalDiscount = manualDiscount + couponDiscount
total         = MAX(0, subtotal + tax - totalDiscount)
```

> **Note:** Tax rate is currently hardcoded to 10%. It is not read from the Settings module.

## Stock Availability

```
availableStock(product, variant?) =
  stockAtWarehouse(product, variant, selectedWarehouseId)
  - reservedQty(product.id, variant?.id)
```

- `stockAtWarehouse` reads from `product.inventory[]` or `variant.inventory[]` matching `warehouseId`
- `reservedQty` is the quantity of the same product/variant already in the cart
- Add to cart is disabled when `availableStock <= 0`
- Cart item "+1" is disabled when `item.quantity >= availableStock`

## Discount Logic

**Manual discount:**
- Fixed: `discount = enteredAmount`
- Percentage: `discount = (subtotal × enteredAmount) / 100`

**Coupon discount:**
- Fixed: `couponDiscount = coupon.discount`
- Percentage: `couponDiscount = (subtotal × coupon.discount) / 100`
- If `coupon.max_discount` is set: `couponDiscount = MIN(couponDiscount, coupon.max_discount)`

Only one coupon can be applied at a time. Manual discount and coupon can both be active simultaneously.

## Cart Behaviour

- Adding the same product/variant again **increments quantity** rather than adding a duplicate row
- Decreasing quantity below 1 **removes the item** from the cart
- On variant product: clicking the card opens a **variant selection modal** first
- Cart ID format: `"{productId}"` for simple products, `"{productId}-{variantId}"` for variants

## Checkout Flow

1. Staff clicks **"Pay Now"**
2. `CheckoutModal` opens — staff selects Cash, Card, or Online
3. If Card: card details form is shown (number, expiry, CVV) — frontend only, not sent to backend
4. Staff clicks **"Confirm"**
5. `isSubmitting = true` — button disabled to prevent double-submit
6. `sellsApi.create(payload)` called
7. On success: `invoiceNo` stored from response, `isSuccessModalOpen = true`
8. On failure: error toast shown, `isSubmitting = false`

## Receipt Printing

- Triggered by "Print Receipt" button in `SuccessModal`
- Opens a new browser window with 80mm thermal receipt HTML
- Calls `window.print()` then closes the window
- Receipt includes: store name, address, date/time, invoice no, customer name, line items, subtotal, tax, discount, total, return policy footer

## Session Reset

After closing `SuccessModal` the session resets:
- Cart cleared
- Customer reset to "Walk-in Customer"
- Manual discount and coupon cleared
- All modals closed
- `isSubmitting = false`

---

# Dependencies

## Internal Modules / APIs

| Module | Usage |
|--------|-------|
| Products | Product catalog and inventory data |
| Customers | Customer list for combobox |
| Coupons | Active coupon list |
| Sells (Orders) | Order creation on checkout |
| Warehouses / Locations | Stock per warehouse |
| Categories | Category tabs for product filtering |

## External Libraries

| Library | Purpose |
|---------|---------|
| `axios` | HTTP requests |
| `shadcn/ui` | UI components (Button, Dialog, Input, Tabs, etc.) |
| `lucide-react` | Icons |
| `sonner` | Toast notifications |
| `tailwind-merge` + `clsx` | Conditional class names |

---

# Security / Permissions

- The page requires an authenticated session (Bearer JWT in localStorage)
- API interceptor auto-attaches token and `company_id` to all requests
- All product, customer, and coupon data is scoped to the authenticated company
- 401 responses trigger a redirect to the login page via the API interceptor
- No role-based access control is implemented at the POS page level — any authenticated user can access it

---

# Error Handling

| Scenario | Handling |
|----------|----------|
| Product load fails | Error toast; products list remains empty |
| Coupon load fails | Error toast; coupon modal shows empty list |
| Checkout API call fails | Error toast; `isSubmitting` reset to `false`; modal stays open |
| Adding out-of-stock item | Add button disabled; not possible via UI |
| Exceeding available stock | +1 button disabled at cart item level |

---

# Testing Notes

- **Product search:** Confirm real-time filter works on name and SKU
- **Category tabs:** Confirm switching categories reloads the correct products
- **Warehouse selector:** Confirm stock levels update when warehouse changes
- **Variant selection:** Confirm variant modal opens for multi-variant products
- **Stock reservation:** Add 3 units of a product with 3 in stock — confirm add button disabled after
- **Cart quantity limits:** Confirm +1 at cart item is disabled when at max available stock
- **Manual discount (fixed):** Apply $10 off — confirm total reduces by $10
- **Manual discount (percentage):** Apply 20% — confirm correct calculation
- **Coupon (fixed):** Apply coupon — confirm discount correct and max_discount cap respected
- **Coupon (percentage):** Apply percentage coupon — confirm correct calculation
- **Walk-in sale:** Complete sale without selecting a customer — confirm order created with `customerName = "Walk-in Customer"`
- **Customer sale:** Select a customer — confirm `customerId` sent in order payload
- **Card payment:** Select Card — confirm card form appears; confirm card details NOT sent to backend
- **Double-submit prevention:** Confirm Pay button disabled while `isSubmitting = true`
- **Success modal:** Confirm invoice number displayed correctly
- **Print receipt:** Confirm print window opens with correct data
- **Session reset:** Confirm cart and discount cleared after success modal closed
- **Reset cart:** Confirm reset dialog fires; confirm cart cleared on confirmation

---

# Open Questions / Missing Details

- **Tax rate hardcoded at 10%:** Should be read from the Settings module (`taxSettings.defaultTaxRate`) rather than hardcoded. This will cause incorrect totals if the business uses a different tax rate.
- **Card details not sent to backend:** The card form is cosmetic only. No payment gateway is integrated. Real card processing (Stripe, Razorpay) is not yet implemented.
- **Coupon validation:** Coupon is applied client-side only (no call to `/coupons/validate`). Expiry, usage limits, and min order amount are not enforced on the frontend before submission.
- **Barcode scanner:** No barcode input support. Products can only be found by typing the name. Adding a barcode/SKU search field would speed up physical retail use.
- **Hold/Draft orders:** No ability to park a transaction and start another.
- **Offline support:** If the backend is unavailable, POS cannot function. No offline/queue mechanism exists.
- **Role-based access:** Any authenticated user can use POS. No permission check against staff roles.
- **Receipt store details:** The receipt currently uses hardcoded store name/address placeholder text. Should read from Settings module (`generalSettings.storeName`, etc.).

---

# Change Log / Notes

- Initial documentation authored from source code analysis of `/home/monsur/Documents/ecommerce-admin`.
- Tax rate noted as hardcoded — flagged for Settings integration.
- Card payment form noted as frontend-only — flagged for payment gateway integration.
- Coupon client-side validation noted — flagged for server-side validation via `/coupons/validate`.
