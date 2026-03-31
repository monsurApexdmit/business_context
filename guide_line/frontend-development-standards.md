# Frontend Development Standards

> **Version 1.0 — Language-Agnostic UI Architecture & Patterns Guide**  
> Engineering Team · 2025  
> _Applicable to any frontend framework: React, Vue, Angular, Svelte, or any other_

---

> ℹ️ All pseudocode examples show **WHAT** to do and the structure to follow. Translate the pattern into your chosen framework — never copy pseudocode literally into production.

---

## Table of Contents

1. [Component Architecture](#1-component-architecture)
2. [Naming Conventions](#2-naming-conventions)
3. [State Management](#3-state-management)
4. [API Integration Layer](#4-api-integration-layer)
5. [Props & Component Contracts](#5-props--component-contracts)
6. [Form Handling](#6-form-handling)
7. [Error & Loading States](#7-error--loading-states)
8. [Routing & Navigation](#8-routing--navigation)
9. [Accessibility (A11Y)](#9-accessibility-a11y)
10. [Performance](#10-performance)
11. [Styling Standards](#11-styling-standards)
12. [Test Cases](#12-test-cases)
13. [Code Review Checklist](#13-code-review-checklist)

---

## 1. Component Architecture

The frontend application must be structured around components. Every piece of UI is a component. Components are composed into larger components. No UI logic lives outside a component.

### 1.1 Component Hierarchy

```
Pages / Screens
      ↓
Layout Components
      ↓
Feature Components
      ↓
UI Components
      ↓
Base / Primitives
```

| Level | Responsibility | Examples |
|---|---|---|
| Pages / Screens | Top-level route views. Composes feature components. Handles route-level data fetching. | `HomePage`, `UserProfilePage`, `CheckoutPage` |
| Layout Components | Structural shells: headers, sidebars, footers, grid wrappers. | `AppLayout`, `DashboardLayout`, `AuthLayout` |
| Feature Components | Domain-specific UI with business logic. Contains state and API calls. | `UserForm`, `OrderList`, `ProductCard` |
| UI Components | Reusable, styled, domain-free interface elements with props. | `Button`, `Modal`, `Table`, `Dropdown`, `Badge` |
| Base / Primitives | The smallest building blocks. No business logic. Pure presentation. | `Text`, `Icon`, `Spinner`, `Divider`, `Avatar` |

### 1.2 Single Responsibility for Components

Each component must do one thing. If a component fetches data AND renders a list AND handles a modal AND validates a form, it must be broken up.

```
# ❌ Avoid — one component doing 6 jobs
COMPONENT UserDashboard:
  // fetches users from API
  // renders user list
  // handles delete confirmation modal
  // manages pagination
  // handles search input
  // formats date display

# ✓ Correct — orchestrates only
COMPONENT UserDashboard:
  RENDER UserSearchBar
  RENDER UserList
  RENDER PaginationControls
  RENDER DeleteConfirmModal
  // Each child handles its own concern
```

### 1.3 Smart vs Dumb Component Pattern

Separate components that know about data (smart/container) from components that only display data (dumb/presentational). This makes UI components reusable and independently testable.

```
// SMART COMPONENT — knows about data, state, API
COMPONENT UserListContainer:
  STATE users = []
  STATE loading = false
  STATE error = null

  ON MOUNT: fetch users from API -> set users

  RENDER UserListView(
    users:    this.users,
    loading:  this.loading,
    error:    this.error,
    onDelete: this.handleDelete
  )

// DUMB COMPONENT — knows nothing about data source
COMPONENT UserListView(users, loading, error, onDelete):
  IF loading: RENDER Spinner
  IF error:   RENDER ErrorMessage(error)
  RENDER users.map(u -> UserRow(u, onDelete))
  // Pure presentation — easily testable, reusable
```

> ⚑ Dumb components must receive all data via props. They must **never** directly call an API, access global state, or import a data store.

---

## 2. Naming Conventions

| Construct | Convention | Example |
|---|---|---|
| Component files | PascalCase | `UserProfileCard.jsx` / `UserProfileCard.vue` |
| Component names | PascalCase | `COMPONENT UserProfileCard` |
| Non-component files | camelCase | `useAuth.js` / `apiClient.js` / `dateUtils.js` |
| Variables & functions | camelCase | `userName` / `fetchOrders()` / `handleSubmit()` |
| Constants | UPPER_SNAKE_CASE | `MAX_FILE_SIZE` / `API_BASE_URL` |
| CSS class names | kebab-case | `user-card` / `nav-link--active` |
| CSS custom properties | kebab-case | `--color-primary` / `--spacing-md` |
| Event handler props | `on` + PascalCase | `onSubmit` / `onDeleteUser` / `onPageChange` |
| Boolean props | `is`/`has`/`can` prefix | `isLoading` / `hasError` / `canEdit` |
| Data-fetching hooks | `use` + Resource | `useUsers()` / `useOrderDetail(id)` |
| Test files | Component.test.ext | `UserCard.test.js` / `useAuth.test.js` |

### 2.1 Naming Anti-Patterns

```
# ❌ Avoid
COMPONENT Data()          // too vague
COMPONENT NewComponent()  // meaningless
handleClick2()            // why 2?
doStuff()                 // what stuff?
temp                      // never commit temp vars
flag                      // flag for what?
data                      // all state is "data"
item                      // item of what?

# ✓ Correct
COMPONENT OrderSummaryCard()
COMPONENT ProductFilterPanel()
handleAddToCart()
calculateDiscountedTotal()
isSubmitting
hasUnsavedChanges
selectedProducts
currentOrderItem
```

---

## 3. State Management

State is the most complex part of any frontend. Poorly managed state causes the majority of UI bugs.

### 3.1 State Classification

| State Type | Definition | Where It Lives |
|---|---|---|
| Local state | UI state relevant only to one component (open/closed, input value) | Inside the component |
| Shared state | State needed by multiple sibling or cousin components | Nearest common parent (lifted state) |
| Global state | App-wide data: authenticated user, theme, locale, permissions | Global store |
| Server state | Data that originates from the API — list of records, single entity | Cache layer (query cache / store slice) |
| URL state | Filters, pagination, selected tabs, search queries | URL query params / route params |
| Form state | In-progress input values, validation errors, dirty/touched flags | Form manager or local state |

### 3.2 State Placement Rules

- Start with local state. Only promote to shared or global when you have a proven need
- Never store derived data in state — compute it from existing state instead
- Never duplicate state — a single source of truth for every piece of data
- Never store server data in global state — use a dedicated server-state / caching layer
- URL state is preferred over local state for anything a user might want to bookmark or share

```
# ❌ Avoid
STATE users = []
STATE userCount = 0       // derived — duplicated!

GLOBAL_STORE.users = await fetchUsers()
// Now you manage staleness manually

GLOBAL_STORE.isDropdownOpen = false
// No other component needs this

# ✓ Correct
STATE users = []
COMPUTED userCount = users.length   // derived, not stored

users = useQuery('users', fetchUsers)
// Auto-refetch, caching, stale handling

COMPONENT Dropdown:
  LOCAL_STATE isOpen = false    // stays local
```

### 3.3 Global Store Structure

The global store must be split into feature slices. Each slice owns its domain data. No slice reads directly from another slice's internals.

```
STORE:
  auth/
    currentUser    : User | null
    permissions    : Permission[]
    isAuthenticated: Boolean

  ui/
    theme          : 'light' | 'dark'
    locale         : String
    notifications  : Notification[]

  // Domain data should live in server-state cache,
  // not in the global store (see Section 4)
```

> ⚠️ Never put API response data (orders, products, users) into the global store. Use a server-state / query cache layer instead.

---

## 4. API Integration Layer

All communication with the backend API must go through a dedicated, centralised API layer. Components must never call `fetch()`, `http`, or `axios` directly.

### 4.1 Layer Structure

```
Component  →  Custom Hook  →  API Service  →  HTTP Client  →  Backend API
```

### 4.2 HTTP Client (Base Layer)

One centralised HTTP client handles base URL, default headers, authentication token injection, and error normalisation.

```
// Centralised HTTP client — configured once
HTTP_CLIENT = createClient({
  baseUrl:    ENV.API_BASE_URL,
  timeout:    10_000,
  headers: {
    'Content-Type': 'application/json',
    'Accept':       'application/json',
  }
})

// Request interceptor — attach auth token
HTTP_CLIENT.onRequest = (config) ->
  token = authStore.getToken()
  IF token: config.headers.Authorization = 'Bearer ' + token
  RETURN config

// Response interceptor — normalise errors
HTTP_CLIENT.onError = (error) ->
  IF error.status == 401: authStore.logout()
  IF error.status == 403: redirect('/forbidden')
  THROW NormalisedApiError(error)
```

### 4.3 API Service (Domain Layer)

Each domain has its own API service file. All endpoint calls for that domain live there. No URL strings exist anywhere except in service files.

```
// userApi.service — all user-related endpoints
FUNCTION getUsers(filters):
  RETURN HTTP_CLIENT.get('/users', params: filters)

FUNCTION getUserById(id):
  RETURN HTTP_CLIENT.get('/users/' + id)

FUNCTION createUser(data):
  RETURN HTTP_CLIENT.post('/users', body: data)

FUNCTION updateUser(id, data):
  RETURN HTTP_CLIENT.patch('/users/' + id, body: data)

FUNCTION deleteUser(id):
  RETURN HTTP_CLIENT.delete('/users/' + id)
```

### 4.4 Custom Hook (Component Interface)

Components interact with the API exclusively through custom hooks. Hooks encapsulate loading state, error state, caching, and retry logic.

```
// useUsers hook — component never knows about HTTP
HOOK useUsers(filters):
  RETURN queryCache.query(
    key:       ['users', filters],
    fetcher:   () -> userApi.getUsers(filters),
    staleTime: 5 minutes,
    retry:     2
  )
  // Returns: { data, isLoading, isError, error, refetch }

// useCreateUser mutation hook
HOOK useCreateUser():
  RETURN queryCache.mutation(
    mutator:   (data) -> userApi.createUser(data),
    onSuccess: () -> queryCache.invalidate(['users']),
    onError:   (err) -> toast.show(err.message, 'error')
  )
  // Returns: { mutate, isLoading, isSuccess, isError }

// Component usage — clean, no HTTP concerns
COMPONENT UserList:
  { data: users, isLoading, isError } = useUsers(filters)
```

> ❌ A component that imports an HTTP client or calls `fetch()` directly is a mandatory refactor. All API calls go through a hook.

---

## 5. Props & Component Contracts

A component's props define its public API. Props must be typed, minimal, and well-named.

### 5.1 Props Rules

- Every prop must have an explicit type — no untyped or `any` props
- Mark optional props explicitly with a default value
- Prefer passing specific typed values over passing entire objects
- Never mutate a prop — treat all props as immutable read-only values
- A component with more than **7 props** is a signal to split it
- Event handler props must always start with `on` (e.g. `onSave`, `onClose`, `onPageChange`)

```
// Component contract — props with types and defaults
COMPONENT UserCard(
  id:        UUID      REQUIRED,
  name:      String    REQUIRED,
  email:     String    REQUIRED,
  avatarUrl: String    OPTIONAL  DEFAULT = null,
  role:      Enum      OPTIONAL  DEFAULT = 'user',
  isActive:  Boolean   OPTIONAL  DEFAULT = true,
  onEdit:    Function  REQUIRED,    // (id) -> void
  onDelete:  Function  OPTIONAL  DEFAULT = null
)
```

### 5.2 Avoid Prop Drilling

Passing props through many layers that do not use them creates tight coupling and makes refactoring painful. Use context/provide-inject instead.

```
# ❌ Avoid — drilled 4 levels deep
PageA(user) ->
  SectionB(user) ->      // B doesn't use user
    PanelC(user) ->      // C doesn't use user
      AvatarD(user.avatarUrl)

# ✓ Correct — context pattern
PageA:
  PROVIDE user = currentUser

AvatarD:
  user = INJECT('user')
  RENDER Avatar(user.avatarUrl)
  // B and C are completely unaware
```

---

## 6. Form Handling

Forms are the primary source of user input and the most common location for bugs. All forms must use a form manager and schema-based validation.

### 6.1 Form Structure Rules

- All form state lives in a form manager — not in component state variables
- Validation is defined in a schema, not in the submit handler
- Validate on **blur** (when a field loses focus) AND on **submit**
- Show field-level error messages directly beneath each input
- Disable the submit button while the form is submitting
- Show a success or error toast/message after submission
- Reset the form to initial state after a successful submission

```
SCHEMA userFormSchema:
  name:     required, string, min=2, max=100
  email:    required, string, format=email
  password: required, string, min=8, pattern=strong

COMPONENT CreateUserForm:
  form = useForm(
    schema:   userFormSchema,
    defaults: { name: '', email: '', password: '' },
    onSubmit: handleCreate
  )

  ASYNC handleCreate(values):
    TRY
      AWAIT createUser(values)
      toast.success('User created successfully')
      form.reset()
    CATCH error
      IF error.type == VALIDATION:
        form.setErrors(error.fields)   // map API errors to fields
      ELSE:
        toast.error(error.message)

  RENDER:
    Input(name,     error: form.errors.name,     ...form.register('name'))
    Input(email,    error: form.errors.email,    ...form.register('email'))
    Input(password, error: form.errors.password, ...form.register('password'))
    Button('Create User', disabled: form.isSubmitting)
```

### 6.2 Form Error Display Rules

- Error messages appear directly under the relevant field — never in a list at the top
- Error messages use the same standard wording as the backend validation guide
- The field input changes to a visual error state (red border, error icon)
- Clear the field error as soon as the user starts correcting it
- Never show raw API error codes or stack traces to the user

---

## 7. Error & Loading States

Every async operation has three states: **loading**, **success**, and **error**. All three must be handled and displayed explicitly.

### 7.1 The Three States Rule

```
COMPONENT UserList:
  { data: users, isLoading, isError, error } = useUsers()

  // State 1: Loading
  IF isLoading:
    RENDER UserListSkeleton()    // skeleton screens, not spinners
    RETURN

  // State 2: Error
  IF isError:
    RENDER ErrorState(
      message: 'Could not load users. Please try again.',
      onRetry: refetch
    )
    RETURN

  // State 3: Success — but handle empty separately
  IF users.length == 0:
    RENDER EmptyState('No users found')
    RETURN

  RENDER users.map(u -> UserRow(u))
```

### 7.2 Loading State Standards

- Use **skeleton screens** (placeholder shapes) for initial data loads — not a spinner
- Use an inline spinner or disabled state for button clicks and form submits
- Never block the full page with a loading overlay for a background operation
- Show loading state within 100ms — never let the UI appear frozen

### 7.3 Error State Standards

- Every error state must offer a recovery action: Retry, Go Back, or Contact Support
- Use human-readable messages — never show HTTP status codes or internal error strings
- Log errors to a monitoring service — never just `console.log` in production

| Scenario | User-Facing Message | Recovery Action |
|---|---|---|
| Network failure | Could not connect. Please check your internet connection. | Retry button |
| 404 Not Found | This record no longer exists or was deleted. | Go Back link |
| 403 Forbidden | You do not have permission to view this. | Contact admin link |
| 500 Server Error | Something went wrong on our end. We have been notified. | Retry / Support link |
| Session expired | Your session has expired. Please sign in again. | Redirect to login |
| Validation (422) | Please correct the highlighted fields and try again. | Scroll to first error |

---

## 8. Routing & Navigation

### 8.1 Route Structure Rules

- All route definitions live in a single centralised route configuration file
- Route paths use kebab-case only — no camelCase or underscores in URLs
- Dynamic segments use descriptive names: `/users/:userId` not `/users/:id`
- Every private route must check authentication before rendering

```
ROUTE CONFIGURATION:

  /                          -> HomePage          [public]
  /auth/login                -> LoginPage         [public, redirect if authed]
  /auth/register             -> RegisterPage      [public, redirect if authed]

  /dashboard                 -> DashboardLayout   [private]
    /dashboard/overview      -> OverviewPage
    /dashboard/users         -> UserListPage
    /dashboard/users/:userId -> UserDetailPage
    /dashboard/settings      -> SettingsPage

  /forbidden                 -> ForbiddenPage     [public]
  /*                         -> NotFoundPage      [public]
```

### 8.2 Route Guards

```
// Auth guard — runs before every private route
GUARD requireAuth(to, from, next):
  IF NOT authStore.isAuthenticated:
    REDIRECT '/auth/login', query: { returnTo: to.path }
    RETURN
  next()

// Permission guard
GUARD requirePermission(permission):
  RETURN (to, from, next) ->
    IF NOT authStore.hasPermission(permission):
      REDIRECT '/forbidden'
      RETURN
    next()

// Applied to routes
  /dashboard/users    -> [requireAuth, requirePermission('users.read')]
  /dashboard/settings -> [requireAuth, requirePermission('settings.manage')]
```

---

## 9. Accessibility (A11Y)

Accessibility is not optional. All features must be usable by keyboard, screen reader, and users with colour vision deficiency. **Accessibility issues block release.**

### 9.1 Mandatory Accessibility Rules

- All interactive elements (buttons, links, inputs) must be reachable and activatable by keyboard
- All images must have descriptive `alt` text — decorative images use `alt=""`
- Form inputs must have an associated visible label or `aria-label`
- Colour must never be the only way to communicate meaning (use icons, text, or patterns too)
- Minimum contrast ratio: **4.5:1** for normal text, **3:1** for large text (WCAG AA)
- Modal dialogs must trap focus and return focus to the trigger on close
- Dynamic content changes must be announced to screen readers via live regions
- Logical heading hierarchy: h1 → h2 → h3, never skip levels

```
// Accessible interactive elements

// Button — always a real button element
ELEMENT button:
  type='button'
  aria-label='Delete user Alice'   // descriptive, not just 'Delete'
  disabled=isDeleting

// Form input with label
ELEMENT label FOR='email-input': 'Email address'
ELEMENT input:
  id='email-input'
  type='email'
  aria-required='true'
  aria-describedby='email-error'
ELEMENT span id='email-error' role='alert': errorMessage

// Loading state announced to screen readers
ELEMENT div aria-live='polite' aria-busy=isLoading:
  IF isLoading: 'Loading users...'
  ELSE: UserList
```

> ❌ A component that uses a `div` or `span` as a clickable element instead of a `button` or `a` tag is an accessibility defect and a code-review blocker.

---

## 10. Performance

### 10.1 Rendering Performance

- Avoid re-rendering components that have not had their data changed — use memoisation where needed
- Never derive expensive computed values inside the render function — memoize them
- Virtualise long lists (100+ items) — never render thousands of DOM nodes at once
- Avoid anonymous functions as event handler props on frequently-rendered components

```
# ❌ Avoid — expensive calc in render (runs every render)
COMPONENT OrderSummary(items):
  RENDER:
    total    = items.reduce(sum prices)    // recalculates every time
    discount = calculateDiscount(items)    // recalculates every time
    RETURN total, discount

# ✓ Correct — memoised
COMPONENT OrderSummary(items):
  total    = MEMO { items.reduce(sum prices) } WHEN [items]
  discount = MEMO { calculateDiscount(items) } WHEN [items]
  RENDER total, discount
```

### 10.2 Code Splitting & Lazy Loading

- Each page/route component must be lazily loaded — never bundle all pages together
- Large feature components (rich text editors, charts, maps) must be lazily imported
- Images must use lazy loading — only load when they enter the viewport
- Third-party scripts must never block the main thread — load them asynchronously

```
// Route-level lazy loading
ROUTES:
  /dashboard/analytics -> LAZY_LOAD( import('./AnalyticsPage') )
  /dashboard/reports   -> LAZY_LOAD( import('./ReportsPage') )

// Heavy component lazy loading
RichTextEditor = LAZY_LOAD( import('./RichTextEditor') )

// Show a fallback while loading
RENDER:
  SUSPENSE fallback=Spinner:
    RichTextEditor
```

### 10.3 Asset Optimisation

- All images must be in a modern format (WebP, AVIF) with a fallback
- Images must declare explicit `width` and `height` to prevent layout shift (CLS)
- SVG icons must be inlined or loaded as a sprite — not as individual HTTP requests
- Fonts must use `font-display: swap` to prevent invisible text while loading

---

## 11. Styling Standards

### 11.1 Design Token System

All visual values — colours, spacing, typography, border radii, shadows — must be defined as design tokens (CSS custom properties or equivalent). Never use raw values in component styles.

```css
/* Design token definitions — single source of truth */

/* Colours */
--color-primary:         #1D4ED8;
--color-primary-hover:   #1E40AF;
--color-danger:          #DC2626;
--color-success:         #16A34A;
--color-text-primary:    #111827;
--color-text-secondary:  #6B7280;
--color-bg-surface:      #FFFFFF;
--color-bg-subtle:       #F9FAFB;
--color-border:          #E5E7EB;

/* Spacing scale */
--space-1: 4px;   --space-2: 8px;   --space-3: 12px;
--space-4: 16px;  --space-6: 24px;  --space-8: 32px;
--space-12: 48px; --space-16: 64px;

/* Typography */
--font-size-sm: 14px;   --font-size-base: 16px;
--font-size-lg: 18px;   --font-size-xl: 20px;
--font-size-2xl: 24px;  --font-size-4xl: 36px;

/* Radii & shadows */
--radius-sm: 4px;  --radius-md: 8px;  --radius-lg: 12px;
--z-modal: 1000;   --z-tooltip: 1100; --z-overlay: 900;
```

### 11.2 Styling Rules

- Never use raw colour hex values or pixel values in component styles — always use tokens
- Never use inline styles for anything other than dynamic, computed values
- Scoped styles per component — global styles only for resets and token definitions
- Responsive design uses the **mobile-first** approach: base styles target mobile, breakpoints extend upward

```
# ❌ Avoid — raw values
STYLE Button:
  background:    #1D4ED8
  padding:       12px 16px
  border-radius: 6px
  font-size:     14px

# ✓ Correct — tokens
STYLE Button:
  background:    var(--color-primary)
  padding:       var(--space-3) var(--space-4)
  border-radius: var(--radius-md)
  font-size:     var(--font-size-sm)
```

---

## 12. Test Cases

Every component and hook must be covered by automated tests. Tests run on every pull request in CI.

### 12.1 Test Types & Coverage Targets

| Type | What It Tests | Target Coverage |
|---|---|---|
| Unit Test | A single utility function, custom hook, or helper in isolation. | ≥ 90% of hooks & utilities |
| Component Test | A component renders correctly and responds to user interactions. | ≥ 80% of all components |
| Integration Test | Multiple components working together: forms, flows, page sections. | ≥ 75% of feature components |
| E2E Test | Full user journey through the browser: login, create, edit, delete. | All critical user flows |
| Accessibility Test | Automated a11y rules: missing labels, contrast, keyboard traps. | 100% of interactive components |
| Snapshot Test | Detects unintended visual regressions in component output. | Key UI components |

### 12.2 Test Naming Convention

```
DESCRIBE 'UserCard component':
  IT 'renders the user name and email'
  IT 'shows the Edit button when onEdit prop is provided'
  IT 'hides the Delete button when onDelete prop is not provided'
  IT 'calls onEdit with the user id when Edit is clicked'
  IT 'shows a loading spinner while isLoading is true'
  IT 'shows an error message when hasError is true'
  IT 'has no accessibility violations'

DESCRIBE 'useUsers hook':
  IT 'returns loading=true on initial fetch'
  IT 'returns users array on successful fetch'
  IT 'returns error on failed fetch'
  IT 'refetches when filters change'
```

### 12.3 Component Test Structure — AAA Pattern

Every test follows the **Arrange → Act → Assert** pattern. Tests simulate real user behaviour — they do not test implementation details.

```
TEST 'calls onDelete with user id when Delete button is clicked':

  // ARRANGE
  mockOnDelete = createMockFunction()
  user = { id: 'user-123', name: 'Alice', email: 'alice@example.com' }
  RENDER UserCard(user: user, onDelete: mockOnDelete)

  // ACT
  deleteButton = screen.getByLabel('Delete user Alice')
  USER_EVENT.click(deleteButton)

  // ASSERT
  ASSERT mockOnDelete WAS_CALLED_ONCE
  ASSERT mockOnDelete WAS_CALLED_WITH 'user-123'
```

### 12.4 What to Test Per Layer

**Component Tests**
- Renders correctly with required props — outputs expected text, structure, and elements
- Renders correctly in all significant visual states: loading, error, empty, populated
- Conditional rendering — shows or hides elements based on props
- User interactions — clicking, typing, selecting triggers the correct callback with correct args
- Does NOT render elements that should be hidden (e.g. delete button when no `onDelete` prop)
- Accessible — no missing labels, buttons are real buttons, inputs have labels

**Hook / Logic Tests**
- Returns correct initial state before any async operation
- Returns `loading=true` while a fetch is in progress
- Returns data correctly after a successful response
- Returns error and correct error message after a failed response
- Triggers a refetch when its dependencies (filters, IDs) change
- Calls the correct API service function with the correct arguments

**Form Tests**
- Submit button is disabled while the form is submitting
- Shows field-level error messages when required fields are empty on submit
- Shows the correct error message for each specific validation rule
- Calls the submit handler with the correct, typed data on valid submission
- Resets all fields to empty / default after successful submission
- Maps API validation errors (422 response) back to the correct form fields

**Route & Navigation Tests**
- Unauthenticated users are redirected to login when accessing a private route
- Authenticated users are redirected away from auth pages
- Users without required permission see the Forbidden page
- 404 routes render the Not Found page

### 12.5 Test Isolation Rules

- Every test is self-contained — no shared mutable state between tests
- Mock all external dependencies: API calls, browser storage, date/time, random values
- Never use real API endpoints in unit or component tests — use mock handlers
- Tests must not depend on execution order — every test must pass when run alone
- Reset all mocks and DOM state before each test
- E2E tests run against a dedicated test environment — never staging or production

> ⚠️ A test that passes alone but fails in a suite has hidden shared state. This is a bug in the test, not the code.

### 12.6 What NOT to Test

- Do not test implementation details — test observable behaviour from the user's perspective
- Do not test framework internals or third-party library behaviour
- Do not test styles or exact CSS values — test structure and content
- Do not use snapshot tests as a substitute for real assertions

```
# ❌ Avoid — testing implementation details (fragile)
TEST 'calls setState':
  ASSERT component.setState WAS_CALLED

TEST 'state.isLoading is true':
  ASSERT component.state.isLoading == true
// Breaks if you rename state variables

# ✓ Correct — testing observable behaviour (resilient)
TEST 'shows spinner while loading':
  ASSERT screen.getByRole('progressbar') EXISTS

TEST 'shows user list after load':
  ASSERT screen.getByText('Alice') IS_VISIBLE
// Works regardless of internal implementation
```

---

## 13. Code Review Checklist

Every pull request touching frontend code must pass all blocker items before approval.

| # | Checklist Item | Blocker? |
|---|---|---|
| 1 | Each component has a single, clearly defined responsibility | ✅ Yes |
| 2 | Dumb components receive all data via props — no direct API calls inside them | ✅ Yes |
| 3 | All component props are explicitly typed with defaults for optional ones | ✅ Yes |
| 4 | All naming follows conventions: PascalCase components, camelCase functions, `on*` handlers | ✅ Yes |
| 5 | No raw API calls in components — all data access goes through hooks | ✅ Yes |
| 6 | No raw `fetch`/`http`/`axios` imports in components or pages | ✅ Yes |
| 7 | All async operations handle loading, error, and empty states explicitly | ✅ Yes |
| 8 | Global store does not contain API response data (use query cache instead) | ✅ Yes |
| 9 | Forms use a form manager and schema-based validation — no manual if-else validation | ✅ Yes |
| 10 | All route transitions go through route guards for private pages | ✅ Yes |
| 11 | All interactive elements are keyboard accessible (real buttons and links) | ✅ Yes |
| 12 | All images have descriptive `alt` text | ✅ Yes |
| 13 | All form inputs have associated labels or `aria-label` | ✅ Yes |
| 14 | No raw hex or pixel values in styles — design tokens used throughout | No |
| 15 | Route/page components are lazily loaded | ✅ Yes |
| 16 | Long lists (100+ items) use virtualisation | ✅ Yes |
| 17 | Component tests cover all visual states and user interactions | ✅ Yes |
| 18 | Form tests verify error messages and success/reset behaviour | ✅ Yes |
| 19 | Tests target observable behaviour, not implementation details | ✅ Yes |
| 20 | No test depends on another test or shared mutable state | ✅ Yes |

---

_Engineering Team · Frontend Development Standards v1.0 · 2025 · Internal Use Only_
