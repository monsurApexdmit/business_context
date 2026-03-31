# Backend API Development Standards

> **Version 1.2 — Language-Agnostic Architecture & Patterns Guide**  
> Engineering Team · 2025  
> _Applicable to any backend language or framework_

---

> ℹ️ All pseudocode examples show **WHAT** to do and the structure to follow. Translate the pattern into the syntax of your chosen language — never copy pseudocode literally into production.

---

## Table of Contents

1. [Architectural Layers](#1-architectural-layers)
2. [SOLID Principles](#2-solid-principles)
3. [Repository Pattern](#3-repository-pattern)
4. [ORM Models & Relation Design](#4-orm-models--relation-design)
5. [N+1 Query Prevention](#5-n1-query-prevention)
6. [DTO — Data Transfer Objects](#6-dto--data-transfer-objects)
7. [Validation & Error Messages](#7-validation--error-messages)
8. [Service Layer](#8-service-layer)
9. [Test Cases](#9-test-cases)
10. [Authentication & Authorisation](#10-authentication--authorisation)
11. [Security Standards](#11-security-standards)
12. [Standard API Response Format](#12-standard-api-response-format)
13. [Pagination, Filtering & Sorting](#13-pagination-filtering--sorting)
14. [API Versioning & Deprecation](#14-api-versioning--deprecation)
15. [Logging & Observability](#15-logging--observability)
16. [Caching Standards](#16-caching-standards)
17. [Database Migrations](#17-database-migrations)
18. [Async Jobs, Queues & Events](#18-async-jobs-queues--events)
19. [Concurrency & Consistency](#19-concurrency--consistency)
20. [Data Privacy & Retention](#20-data-privacy--retention)
21. [Naming Conventions](#21-naming-conventions)
22. [Code Review Checklist](#22-code-review-checklist)

---

## 1. Architectural Layers

All backend APIs must follow a strict layered architecture. Each layer has a single responsibility. Layers communicate only with their direct neighbour — no layer may skip or bypass another.

```
Route / Endpoint  →  Controller  →  Service  →  Repository  →  Database / ORM
```

| Layer | Single Responsibility | Must NEVER Contain |
|---|---|---|
| Route / Endpoint | Declare URL paths, attach middleware, forward to controller | Logic, DB calls, business rules |
| Controller | Parse request → call service → format and return response | Business logic, DB queries, ORM imports |
| Service | All business logic, domain rules, orchestration | HTTP objects (request/response), DB queries |
| Repository | All database operations — read, write, update, delete | Business logic, HTTP concerns, validation |
| Database / ORM | Schema definition, column types, relation mapping | Logic of any kind |

> ❌ A controller must **NEVER** import a repository. A service must **NEVER** reference an HTTP request or response object.

---

## 2. SOLID Principles

Every class and module must be designed around the five SOLID principles. These are language-independent design rules. A violation is a mandatory code-review blocker.

### 2.1 Single Responsibility Principle (SRP)

A class or module must have exactly one reason to change. If you describe its job using the word "and", it is violating SRP — split it.

```
# ❌ Avoid
CLASS UserService:
  METHOD register()    // business logic
  METHOD sendEmail()   // email concern
  METHOD saveToDb()    // database concern
  // 3 responsibilities = 3 reasons to change

# ✓ Correct
CLASS UserService:
  METHOD register()    // business logic only

CLASS EmailService:
  METHOD sendWelcome()

CLASS UserRepository:
  METHOD save()
```

### 2.2 Open / Closed Principle (OCP)

A class must be open for extension but closed for modification. New behaviour is added by creating new classes — not by editing existing ones.

```
# ❌ Avoid
CLASS PaymentService:
  METHOD process(type):
    IF type == 'stripe': ...
    IF type == 'paypal': ...
    // Every new provider = edit this file

# ✓ Correct
INTERFACE IPaymentProvider:
  METHOD charge(amount)

CLASS StripeProvider IMPLEMENTS IPaymentProvider
CLASS PaypalProvider IMPLEMENTS IPaymentProvider
CLASS BkashProvider  IMPLEMENTS IPaymentProvider
// New provider = new class, zero edits
```

### 2.3 Liskov Substitution Principle (LSP)

Any subclass must be fully substitutable for its parent class without breaking the program. Never override a method just to throw "not implemented" — that is a signal to redesign the abstraction.

```
# ❌ Avoid
CLASS ReadOnlyRepository EXTENDS Repository:
  METHOD save():
    THROW 'Not supported'
// Breaks LSP — caller cannot substitute safely

# ✓ Correct
INTERFACE IReadRepository:
  METHOD findById(id)
  METHOD findAll()

INTERFACE IWriteRepository:
  METHOD save(data)
  METHOD delete(id)
// Segregated — no forced overrides
```

### 2.4 Interface Segregation Principle (ISP)

No class should be forced to implement methods it does not use. Prefer many small focused interfaces over one large general-purpose interface.

```
# ❌ Avoid
INTERFACE IRepository:
  findAll()
  findById(id)
  save(data)
  delete(id)
  bulkImport(list)   // not all repos need this
  exportToCsv()      // not all repos need this

# ✓ Correct
INTERFACE IReadRepository:
  findAll()
  findById(id)

INTERFACE IWriteRepository:
  save(data)
  delete(id)

INTERFACE IBulkRepository:
  bulkImport(list)
```

### 2.5 Dependency Inversion Principle (DIP)

High-level modules must not depend on low-level concrete classes. Both must depend on abstractions (interfaces). Dependencies are injected from the outside — never instantiated inside a class.

```
# ❌ Avoid
CLASS OrderService:
  CONSTRUCTOR():
    this.repo = NEW OrderRepository()
    // Hard-coded — untestable, tightly coupled

# ✓ Correct
CLASS OrderService:
  CONSTRUCTOR(repo: IOrderRepository):
    this.repo = repo
    // Injected — swappable, easily mocked
```

> ⚑ Never use `new` inside a service or controller for infrastructure classes (repositories, email senders, HTTP clients). Always inject them.

---

## 3. Repository Pattern

The Repository Pattern is a data-access abstraction. The service layer talks only to an interface — it has no knowledge of which database, ORM, or query language is being used underneath. This makes the business layer fully testable and portable.

### 3.1 Repository Interface (Contract)

Define the interface first. The service layer depends only on this contract, never on a concrete implementation.

```
INTERFACE IUserRepository:

  FUNCTION findById(id)         -> User | null
  FUNCTION findByEmail(email)   -> User | null
  FUNCTION findAll(filters)     -> PaginatedResult<User>
  FUNCTION create(data)         -> User
  FUNCTION update(id, data)     -> User
  FUNCTION delete(id)           -> void
```

### 3.2 Concrete Repository Implementation

The concrete class implements the interface using whatever ORM or query builder the project uses. All database queries must live exclusively inside repository classes.

```
CLASS UserRepository IMPLEMENTS IUserRepository:

  FUNCTION findById(id):
    RETURN db.users.findOne(
      WHERE id = id,
      INCLUDE [roles, address]   // eager-load relations here
    )

  FUNCTION findAll(filters):
    SET page  = filters.page  OR 1
    SET limit = filters.limit OR 20
    SET rows, total = db.users.findAndCount(
      WHERE  filters.search ? name LIKE '%search%' : ALL,
      LIMIT  limit,
      OFFSET (page - 1) * limit,
      ORDER  createdAt DESC
    )
    RETURN { data: rows, total, page, limit }

  FUNCTION create(data):
    RETURN db.users.insert(data)
```

> ❌ A query written inside a service, controller, or anywhere outside a repository class is a mandatory refactor blocker.

---

## 4. ORM Models & Relation Design

### 4.1 Model Definition Rules

- Every column must declare its data type explicitly — no implicit mappings
- Every column must declare whether it is nullable or required
- All models must have `createdAt` and `updatedAt` timestamp columns
- Use soft deletes via a `deletedAt` column — never hard-delete rows from primary tables
- Use UUID primary keys, not auto-increment integers, for distributed safety
- Never embed business logic inside a model class — models are schema only

```
MODEL User:
  id          UUID          PRIMARY KEY   DEFAULT generate_uuid()
  name        STRING(100)   NOT NULL
  email       STRING(255)   NOT NULL      UNIQUE
  password    STRING        NOT NULL
  role        ENUM          NOT NULL      DEFAULT 'user'
  isActive    BOOLEAN       NOT NULL      DEFAULT true
  deletedAt   TIMESTAMP     NULLABLE      DEFAULT null
  createdAt   TIMESTAMP     NOT NULL      AUTO
  updatedAt   TIMESTAMP     NOT NULL      AUTO
```

### 4.2 Relation Declaration Rules

- All relations must be declared explicitly on both sides of the association
- Always assign a readable alias to each relation
- Specify the foreign key explicitly — never rely on ORM convention guessing
- Never infer or resolve a relation at query time without declaring it on the model

```
// One-to-Many
User       HAS MANY    Orders    (foreignKey: userId,    as: 'orders')
Order      BELONGS TO  User      (foreignKey: userId,    as: 'user')

// One-to-Many nested
Order      HAS MANY    OrderItem (foreignKey: orderId,   as: 'items')
OrderItem  BELONGS TO  Order     (foreignKey: orderId,   as: 'order')
OrderItem  BELONGS TO  Product   (foreignKey: productId, as: 'product')

// Many-to-Many
Product    BELONGS TO MANY  Category  (through: ProductCategories, as: 'categories')
Category   BELONGS TO MANY  Product   (through: ProductCategories, as: 'products')
```

---

## 5. N+1 Query Prevention

An N+1 query defect occurs when the code fetches a list of records and then issues a separate database query for each record to load its related data. This silently multiplies database load and is a critical performance issue.

### 5.1 The Problem

```
// ❌ N+1 Anti-Pattern
// 1 query to load 100 orders
orders = repository.findAll()

// Then 1 query PER order to load the user = 100 extra queries
FOR EACH order IN orders:
  order.user = repository.findUserById(order.userId)

// Total: 101 queries instead of 2
// With 500 orders this becomes 501 queries — invisible but catastrophic
```

### 5.2 Fix — Eager Loading

Load all required related data in the same query using joins or sub-selects. This is the preferred solution for predictable, fixed relations.

```
// ✓ Eager Loading
orders = repository.findAll(
  INCLUDE user,
  INCLUDE items -> INCLUDE product
)
// Result: 2-3 total queries regardless of how many orders exist
```

### 5.3 Fix — Batch Loading

For cases where a JOIN is not practical, collect all foreign key IDs first and resolve them in a single query. This is the DataLoader pattern.

```
// ✓ Batch Loading Pattern
// Step 1: Load the base list
orders = repository.findAll(filters)

// Step 2: Collect all unique foreign keys
userIds = UNIQUE( orders.map(o -> o.userId) )

// Step 3: One query for all related records
users   = userRepository.findByIds(userIds)
userMap = MAP( users, key: user.id, value: user )

// Step 4: Merge in memory — zero extra DB calls
FOR EACH order IN orders:
  order.user = userMap[order.userId]
```

| Approach | Best For | Total Queries |
|---|---|---|
| Eager loading (include/join) | Fixed, known nested relations in a list endpoint | 2–3 queries total |
| Batch loading (collect + one query) | Dynamic or graph-style relations | 1 per relation type |
| Loop + individual query (N+1) | **Never — always forbidden in production code** | 1 + N per record |

> ⚑ Every repository method that returns a list must be reviewed for N+1. Eager-load or batch-load all associations — never load relations inside a loop.

---

## 6. DTO — Data Transfer Objects

A DTO (Data Transfer Object) is a plain, typed structure that defines the exact shape of data crossing a layer boundary. DTOs decouple the internal database model from the external API contract and protect sensitive fields from leaking to clients.

### 6.1 DTO Types

| DTO Type | Direction | Purpose |
|---|---|---|
| Request DTO | Client → Controller | Defines the allowed shape of incoming payload |
| Response DTO | Service → Client | Controls exactly which fields the client receives |
| Service DTO | Controller → Service | Carries validated, clean data into the business layer |
| Persistence DTO | Service → Repository | Shape of data passed to the DB for create or update |

### 6.2 DTO Definitions

```
// Request DTO — what the client sends
STRUCTURE CreateUserRequest:
  name     : String    REQUIRED  minLength=2  maxLength=100
  email    : String    REQUIRED  format=email
  password : String    REQUIRED  minLength=8
  role     : Enum      OPTIONAL  values=[admin, user, moderator]  default=user

// Response DTO — what the client gets back
// password, deletedAt, internalScore are intentionally absent
STRUCTURE UserResponse:
  id        : UUID
  name      : String
  email     : String
  role      : String
  createdAt : Timestamp

// Mapper function — converts ORM model to Response DTO
FUNCTION toUserResponse(user: UserModel) -> UserResponse:
  RETURN {
    id:        user.id,
    name:      user.name,
    email:     user.email,
    role:      user.role,
    createdAt: user.createdAt.toISO()
  }
```

### 6.3 DTO Rules

- Never return a raw ORM model object directly to the client
- Never accept a raw ORM model as input to a service method
- DTOs must be plain data structures — no methods, no ORM decorators
- Every DTO field must have an explicit type — no untyped or `any` fields
- Response DTOs must exclude all sensitive fields: passwords, tokens, internal flags
- Request DTOs are validated before they enter the service layer

```
# ❌ Avoid — Raw ORM model returned directly
FUNCTION getUser(id):
  user = this.repo.findById(id)
  RETURN user
  // Exposes: password, deletedAt, internalScore, raw relations

# ✓ Correct — Mapped to clean Response DTO
FUNCTION getUser(id):
  user = this.repo.findById(id)
  IF NOT user:
    THROW NotFoundError()
  RETURN toUserResponse(user)
  // Only exposes declared safe fields
```

---

## 7. Validation & Error Messages

All incoming data must be validated at the API boundary before it reaches any service or repository. Validation is declared in a dedicated **Request file** for each operation — it is never written inline inside controllers or services.

### 7.1 Where Validation Runs

```
Client Request  →  Request File [VALIDATE HERE]  →  Controller  →  Service  →  Repository
```

Each endpoint has a corresponding Request file that owns all validation rules for that operation. The controller only ever receives data that has already been validated and typed by the Request file.

### 7.2 Request File Structure

A Request file is responsible for:
- Declaring all field-level validation rules for one specific request (e.g. `CreateUserRequest`, `UpdateProductRequest`)
- Returning structured error messages when rules are violated
- Stripping unknown fields before passing data to the controller

```
REQUEST FILE: CreateUserRequest

  name:
    required:  true
    type:      string
    minLength: 2
    maxLength: 100
    message:   'name is required and must be 2–100 characters'

  email:
    required:  true
    type:      string
    format:    email
    message:   'email must be a valid email address'

  password:
    required:  true
    type:      string
    minLength: 8
    pattern:   must contain 1 uppercase, 1 digit, 1 symbol
    message:   'password must be at least 8 characters with 1 uppercase, 1 number, and 1 symbol'

  role:
    required:  false
    type:      enum
    values:    [admin, user, moderator]
    default:   user
    message:   'role must be one of: admin, user, moderator'

  STRIP_UNKNOWN_FIELDS: true   // always remove fields not in schema
```

> One Request file per operation. `CreateUserRequest` and `UpdateUserRequest` are separate files even if they share most rules — update requests typically mark all fields optional.

### 7.3 Naming Convention

| Operation | Request File Name |
|---|---|
| Create resource | `CreateProductRequest` |
| Full update | `UpdateProductRequest` |
| Partial update (PATCH) | `PatchProductRequest` |
| Query / filter | `ListProductsRequest` |
| Action | `VerifyEmailRequest`, `ResetPasswordRequest` |

### 7.4 Validation Error Message Standards

| Rule Violated | ❌ Bad Message | ✓ Correct Message |
|---|---|---|
| Required field | `"Invalid input"` | `"name is required"` |
| Type mismatch | `"Bad value"` | `"age must be a number"` |
| Min length | `"Too short"` | `"password must be at least 8 characters"` |
| Max length | `"Too long"` | `"name must not exceed 100 characters"` |
| Email format | `"Wrong format"` | `"email must be a valid email address"` |
| Enum value | `"Not allowed"` | `"role must be one of: admin, user, moderator"` |
| Min value | `"Out of range"` | `"quantity must be at least 1"` |
| Max value | `"Out of range"` | `"quantity must not exceed 999"` |
| Pattern mismatch | `"Invalid password"` | `"password must contain 1 uppercase, 1 number, and 1 symbol"` |
| Date format | `"Wrong date"` | `"startDate must be a valid ISO 8601 date (e.g. 2025-06-15)"` |
| Unique conflict | `"Error"` | `"email address is already registered"` |
| Cross-field rule | `"Invalid"` | `"endDate must be after startDate"` |

### 7.5 Error Response Shape

```json
// HTTP 422 Unprocessable Entity
{
  "success":    false,
  "statusCode": 422,
  "message":    "Validation failed. Please correct the errors below.",
  "errors": [
    { "field": "email",    "message": "email must be a valid email address" },
    { "field": "password", "message": "password is required" },
    { "field": "role",     "message": "role must be one of: admin, user, moderator" }
  ],
  "traceId": "req_7f3a9c12b"
}
```

### 7.6 Validation Rules

- **All validation lives in the Request file** — never in the controller, service, or repository
- Validate request body, query parameters, and path parameters with equal rigor
- Strip unknown fields — never pass unrecognised input into the service layer
- On update endpoints (PATCH), create a separate Request file with all fields marked optional
- Business-rule violations (e.g. 'email already taken') return **409 Conflict**, not 422 — these are checked in the service layer, not the Request file
- Services must still enforce domain invariants even after the Request file validation passes

> ⚑ Never return a single generic error. Collect **ALL** field errors and return them together in one response.

---

## 8. Service Layer

### 8.1 What Belongs in a Service

- Business logic — rules, conditions, calculations
- Orchestration — calling multiple repositories or external services in sequence
- Domain validation — invariants beyond simple field checks
- Transaction management — wrapping multi-table writes in an atomic transaction
- Event dispatch — publishing domain events after state changes

### 8.2 Transaction Rule

Any operation that writes to more than one table must be wrapped in a database transaction. If any step fails, all writes roll back together. Partial writes are a data-integrity defect.

```
FUNCTION createOrderWithItems(dto):

  BEGIN TRANSACTION:

    order = orderRepository.create(
      { userId: dto.userId, total: dto.total },
      transaction: trx
    )

    orderItemRepository.bulkCreate(
      dto.items.map(item -> { ...item, orderId: order.id }),
      transaction: trx
    )

    inventoryRepository.decrementStock(
      dto.items,
      transaction: trx
    )

    RETURN toOrderResponse(order)

  // If any step throws, ALL writes above are rolled back
  ON ERROR: ROLLBACK, RE-THROW
```

> ❌ A service that calls `create()` and `update()` on different tables without a transaction is a data-integrity defect.

---

## 9. Test Cases

Testing is not optional. Every layer must have automated tests. All tests must be independent — no test may depend on another's state.

### 9.1 Test Types & Coverage Targets

| Type | What It Tests | Target Coverage |
|---|---|---|
| Unit Test | A single function or class in isolation. All dependencies mocked. | ≥ 90% of services & utilities |
| Integration Test | Two or more layers together (e.g. service + real repository + DB). | ≥ 75% of repositories |
| Contract Test | Validates that the API response matches its declared DTO shape. | 100% of public endpoints |
| E2E Test | Full HTTP request → DB → response flow through a running server. | All critical user journeys |

### 9.2 Test Naming Convention

```
DESCRIBE 'UserService':
  DESCRIBE 'createUser':
    IT 'should return a UserResponse DTO on valid input'
    IT 'should throw ValidationError when email is missing'
    IT 'should throw ConflictError when email is already registered'
    IT 'should store a hashed password, never plain text'
    IT 'should assign the default role of user when role is not provided'
    IT 'should emit a UserCreated event after successful creation'

  DESCRIBE 'getUserById':
    IT 'should return a UserResponse DTO when user exists'
    IT 'should throw NotFoundError when user does not exist'
    IT 'should not expose password or deletedAt in the response'
```

### 9.3 Unit Test Structure — AAA Pattern

Every unit test must follow the **Arrange → Act → Assert** pattern.

```
TEST 'should return UserResponse DTO on valid input':

  // ARRANGE — set up inputs and mock dependencies
  input = {
    name:     'Alice',
    email:    'alice@example.com',
    password: 'Secure@123'
  }
  mockRepo.create = MOCK returning savedUser
  mockHasher.hash = MOCK returning 'hashed_password'

  // ACT — call the unit under test
  result = userService.createUser(input)

  // ASSERT — verify the outcome
  ASSERT result.id        IS_NOT null
  ASSERT result.email     EQUALS 'alice@example.com'
  ASSERT result.password  IS_NOT_IN result   // must not be exposed
  ASSERT mockRepo.create  WAS_CALLED_ONCE
  ASSERT mockHasher.hash  WAS_CALLED_WITH input.password
```

### 9.4 What to Test Per Layer

**Service Layer**
- Happy path — correct input returns correct DTO output
- Error paths — each invalid condition throws the expected error type
- Edge cases — empty lists, null fields, boundary values
- Side effects — events emitted, emails triggered, external calls made
- Security — sensitive fields are absent from the returned DTO

**Repository / Integration Tests**
- `findById` returns the correct record with eager-loaded relations
- `findAll` paginates correctly and respects filters
- `create` persists all fields to the database
- `update` modifies only the specified fields
- `delete` soft-deletes and excludes the record from subsequent queries
- Transactions roll back fully when any step throws

**Controller / Contract Tests**
- Returns the correct HTTP status code for each outcome
- Response body matches the declared Response DTO shape exactly
- Returns 422 with a field-level errors array on invalid input
- Returns 401 when no authentication token is provided
- Returns 403 when the token lacks the required permission
- Does NOT expose internal fields in any response

### 9.5 Test Isolation Rules

- Each test must be fully self-contained — no shared state between tests
- Unit tests must mock all external dependencies
- Integration tests run against a dedicated test database — never staging or production
- The test database must be reset to a clean state before each test suite
- Tests must not depend on execution order
- No test may use real credentials, real API keys, or real third-party services

### 9.6 Coverage Requirements

```
Service layer (business logic)    >=  90%
Controller layer                  >=  80%
Repository layer                  >=  75%
Utility / helper functions        >=  85%
Validation schemas                >= 100%
Overall project minimum           >=  80%

Critical paths — MUST have 100% coverage:
  - Authentication and token validation
  - Permission / authorisation checks
  - Any function that writes to the database
  - Any function that sends an email or notification
  - Any function that charges money or modifies balances
```

---

## 10. Authentication & Authorisation

### 10.1 Token Standard

- Use JWT with asymmetric signing (RS256 or ES256) — never HS256 in production
- Access token expiry: **15 minutes** maximum
- Refresh token expiry: **7 days**; rotated on every use
- Store refresh tokens in HttpOnly, Secure, SameSite=Strict cookies — never in localStorage
- Never embed sensitive data (password, card number, PII) inside a JWT payload

```
// JWT payload structure
ACCESS_TOKEN payload:
  sub:         'user-uuid'          // subject — user ID only
  role:        'admin'
  permissions: ['users.read', ...]
  iat:         unix_timestamp       // issued at
  exp:         iat + 900            // expires in 15 min

// Never include:
  password, email, card_number, ssn, internal_flags
```

### 10.2 Refresh Token Flow

```
ENDPOINT POST /auth/refresh:
  refreshToken = READ from HttpOnly cookie
  IF NOT refreshToken: THROW UnauthorizedError()

  storedToken = tokenRepository.findByToken(hash(refreshToken))
  IF NOT storedToken OR storedToken.isRevoked: THROW UnauthorizedError()
  IF storedToken.expiresAt < NOW(): THROW UnauthorizedError('Session expired')

  // Rotate — invalidate old, issue new
  tokenRepository.revoke(storedToken.id)
  newRefreshToken = generateSecureToken()
  tokenRepository.save({ userId, token: hash(newRefreshToken), expiresAt: NOW + 7d })

  newAccessToken = jwt.sign(userPayload, privateKey, expiresIn: '15m')

  SET_COOKIE 'refreshToken' = newRefreshToken, HttpOnly, Secure, SameSite=Strict
  RETURN { accessToken: newAccessToken }
```

### 10.3 Role-Based Access Control (RBAC)

```
// Permission structure
PERMISSIONS:
  users.read    users.create    users.update    users.delete
  orders.read   orders.create   orders.update   orders.delete
  reports.view  settings.manage

// Roles are permission bundles
ROLE admin:      ALL permissions
ROLE manager:    users.read, orders.*, reports.view
ROLE staff:      orders.read, orders.create
ROLE viewer:     *.read only

// Endpoint guards — declare required permission, not role
ROUTE GET  /users       [requireAuth, requirePermission('users.read')]
ROUTE POST /users       [requireAuth, requirePermission('users.create')]
ROUTE DELETE /users/:id [requireAuth, requirePermission('users.delete')]
```

### 10.4 Ownership Checks

```
# ❌ Avoid — permission check only
FUNCTION getOrder(id, currentUser):
  IF NOT currentUser.can('orders.read'):
    THROW ForbiddenError()
  order = orderRepo.findById(id)
  RETURN order  // Returns ANY user's order!

# ✓ Correct — permission + ownership
FUNCTION getOrder(id, currentUser):
  IF NOT currentUser.can('orders.read'):
    THROW ForbiddenError()
  order = orderRepo.findById(id)
  IF NOT order: THROW NotFoundError()
  IF order.userId != currentUser.id:
    THROW ForbiddenError()
  RETURN toOrderResponse(order)
```

> ⚠️ Admin roles may bypass ownership checks. Document every bypass explicitly in code with a comment.

---

## 11. Security Standards

### 11.1 Password Handling

- Hash with **bcrypt** (cost=12), **argon2**, or **scrypt** — never MD5, SHA-1, or plain SHA-256
- Minimum work factor: bcrypt cost=12, argon2 memory=64MB
- Never log passwords, even partially
- Password reset tokens must be cryptographically random, single-use, expire in 1 hour

```
// ✓ Correct
FUNCTION register(dto):
  hashedPassword = bcrypt.hash(dto.password, cost=12)
  user = userRepo.create({ ...dto, password: hashedPassword })
  RETURN toUserResponse(user)  // password not in response DTO

// ❌ Never do this
user.password == inputPassword          // plain text compare
sha256(password) == storedHash          // weak hashing
logger.info('Login: ' + password)       // logging password
```

### 11.2 Secret Management

- All secrets must live in environment variables — never in code
- Use a secrets manager in production (AWS Secrets Manager, Vault, GCP Secret Manager)
- `.env` files are for local development only — never commit them
- Rotate all secrets immediately if accidentally committed
- Validate all required environment variables at startup — fail fast if any are missing

```
FUNCTION validateConfig():
  required = ['DATABASE_URL', 'JWT_PRIVATE_KEY', 'REDIS_URL', 'SMTP_HOST']
  missing = required.FILTER(key -> ENV[key] IS EMPTY)
  IF missing.length > 0:
    LOG.fatal('Missing required env vars: ' + missing.join(', '))
    PROCESS.EXIT(1)  // Never start with missing config
```

### 11.3 Injection & Input Protection

- Use parameterised queries — never concatenate user input into SQL
- Sanitise user input before storing — strip HTML from non-HTML fields
- Validate file upload MIME types and extensions — never trust client-declared type
- Reject file uploads larger than defined maximum (default: 5 MB)
- Never pass user-supplied URLs to server-side HTTP calls without whitelist validation (SSRF)

### 11.4 Security Headers & CORS

- CORS must explicitly whitelist allowed origins — never use wildcard `*` in production
- Apply security headers on every response: `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`
- HTTPS only — HTTP requests must redirect to HTTPS

### 11.5 Rate Limiting & Brute-Force Protection

```
RATE_LIMIT:
  global:              100 req / min  per IP
  POST /auth/login:      5 req / 15min per IP
  POST /auth/register:   3 req / hour  per IP
  POST /auth/reset:      3 req / hour  per email
  GET  /export/*:       10 req / hour  per user

ON LIMIT EXCEEDED:
  RETURN 429 Too Many Requests
  HEADER Retry-After: seconds_until_reset
```

---

## 12. Standard API Response Format

Every API response must use the same envelope structure. Clients must be able to rely on a completely predictable shape at all times.

### 12.1 Success Response

```json
// Single resource — HTTP 200
{
  "success":    true,
  "statusCode": 200,
  "message":    "User retrieved successfully",
  "data":       { "...userResponseDTO": "..." },
  "traceId":    "req_7f3a9c12b"
}

// List with pagination — HTTP 200
{
  "success":    true,
  "statusCode": 200,
  "message":    "Users retrieved successfully",
  "data":       ["...userResponseDTOs"],
  "meta": {
    "page":         1,
    "limit":        20,
    "totalRecords": 150,
    "totalPages":   8,
    "hasNextPage":  true,
    "hasPrevPage":  false
  },
  "traceId": "req_7f3a9c12b"
}

// Created resource — HTTP 201
{
  "success":    true,
  "statusCode": 201,
  "message":    "User created successfully",
  "data":       { "...newUserResponseDTO": "..." },
  "traceId":    "req_7f3a9c12b"
}
```

### 12.2 Error Response

```json
{
  "success":    false,
  "statusCode": 404,
  "message":    "User not found",
  "traceId":    "req_7f3a9c12b"
}
```

> ❌ Never expose stack traces, file paths, query strings, or internal error codes in API responses.

---

## 13. Pagination, Filtering & Sorting

Every list endpoint must support pagination, filtering, and sorting using a consistent query parameter convention.

### 13.1 Standard Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | Integer | 1 | Page number, 1-based |
| `limit` | Integer | 20 | Records per page. Maximum: 100 |
| `sortBy` | String | createdAt | Column name to sort by |
| `sortOrder` | Enum | desc | Sort direction: `asc` or `desc` |
| `search` | String | — | Full-text search across default searchable fields |
| `filter[x]` | String | — | Filter by specific field, e.g. `filter[status]=active` |
| `cursor` | String | — | Cursor for cursor-based pagination |

```
// Example requests
GET /users?page=2&limit=10
GET /orders?sortBy=total&sortOrder=asc&filter[status]=pending
GET /products?search=laptop&filter[categoryId]=cat-123&limit=20

// Must reject with 400
GET /users?limit=500        // exceeds maximum
GET /users?sortBy=password  // sensitive field — whitelist sort columns
```

### 13.2 Offset vs Cursor Pagination

| Type | Best For | Drawback |
|---|---|---|
| Offset (page + limit) | Admin dashboards, small-medium datasets, random page access | Skips or duplicates records if data changes between pages |
| Cursor (cursor token) | Feeds, timelines, large or fast-changing datasets | Cannot jump to an arbitrary page number |

> ⚑ Sort column must be validated against a whitelist. Never allow sorting by a column name taken directly from the query string.

---

## 14. API Versioning & Deprecation

### 14.1 Versioning Strategy

- Use URL path versioning: `/api/v1/resource`
- Increment the major version number only on **breaking changes**
- Non-breaking additions do not require a new version
- At most two major versions may be active simultaneously

```
// Breaking change = new version required
Breaking:     removing a field, renaming a field, changing a field type,
              changing an HTTP method, changing error response shape

Non-breaking: adding a new optional response field,
              adding a new endpoint,
              adding a new optional query parameter
```

### 14.2 Deprecation Policy

- Announce via a `Deprecation` response header on affected endpoints
- Minimum deprecation notice period: **90 days** before removal
- After 90 days, return `410 Gone` — never silently break

```
// Deprecation headers
RESPONSE headers:
  Deprecation:  'true'
  Sunset:       'Sat, 30 Nov 2025 00:00:00 GMT'
  Link:         '/api/v2/users; rel=successor-version'
```

---

## 15. Logging & Observability

### 15.1 Log Levels

| Level | When to Use |
|---|---|
| FATAL | Application cannot start or continue — crash, unrecoverable state |
| ERROR | Operation failed — unhandled exceptions, downstream service failures |
| WARN | Recoverable issue — deprecated usage, approaching limits, retrying |
| INFO | Lifecycle events — service start/stop, request received, job completed |
| DEBUG | Detailed diagnostic data — disabled in production |
| TRACE | Fine-grained step tracing — development only |

### 15.2 Structured Log Format

```json
{
  "timestamp":   "2025-06-15T10:23:44.123Z",
  "level":       "INFO",
  "service":     "order-service",
  "version":     "2.4.1",
  "traceId":     "req_7f3a9c12b",
  "userId":      "usr_abc123",
  "method":      "POST",
  "path":        "/api/v1/orders",
  "statusCode":  201,
  "durationMs":  142,
  "message":     "Order created",
  "meta": {
    "orderId":   "ord_xyz789",
    "itemCount": 3
  }
}

// NEVER log these fields:
//   password, token, refreshToken, cardNumber, cvv, ssn, secret
```

### 15.3 Trace / Correlation IDs

```
MIDDLEWARE attachTraceId(request, response, next):
  traceId = request.headers['X-Trace-ID'] OR generateUUID()
  request.traceId = traceId
  response.setHeader('X-Trace-ID', traceId)
  logger.setContext({ traceId })
  next()

// Propagate to downstream services
HTTP_CLIENT.onRequest = (config) ->
  config.headers['X-Trace-ID'] = currentRequest.traceId
  RETURN config
```

### 15.4 Health & Readiness Endpoints

```
GET /health:
  RETURN 200 {
    "status":  "ok",
    "version": APP_VERSION,
    "uptime":  process.uptime()
  }

GET /ready:
  db_ok    = database.ping()
  redis_ok = cache.ping()
  IF NOT db_ok OR NOT redis_ok:
    RETURN 503 { status: 'unavailable', checks: { db: db_ok, redis: redis_ok } }
  RETURN 200 { status: 'ready' }
```

### 15.5 Metrics & Alerting

- Track: request rate, error rate, p95/p99 latency, DB query time, queue depth
- Alert on: error rate > 1% over 5 minutes, p99 latency > 2 seconds, failed job rate > 5%

---

## 16. Caching Standards

### 16.1 When to Cache

| Cache This | Do NOT Cache This |
|---|---|
| Configuration and lookup data (rarely changes) | User-specific sensitive data (bank balance, orders) |
| Public list data (product catalogue, categories) | Data that must be real-time (inventory stock count) |
| Computed aggregates (dashboards, counts) | Authentication tokens — use a token store instead |
| External API responses with a known refresh cycle | Data modified by the same request |

### 16.2 Cache Rules

- Every cached item must have an explicit TTL — never cache without expiry
- Cache keys must be deterministic and include all parameters that affect the result
- The service that writes the data owns cache invalidation
- Use cache-aside pattern: check cache first, on miss fetch from DB and populate cache

```
FUNCTION getProductById(id):
  cacheKey = 'product:' + id

  cached = cache.get(cacheKey)
  IF cached: RETURN cached

  product = productRepo.findById(id)
  IF NOT product: THROW NotFoundError()

  cache.set(cacheKey, toProductResponse(product), TTL: 300)  // 5 min
  RETURN toProductResponse(product)

// Invalidation — called after any write
FUNCTION updateProduct(id, data):
  updated = productRepo.update(id, data)
  cache.delete('product:' + id)
  cache.delete('products:list:*')
  RETURN toProductResponse(updated)
```

---

## 17. Database Migrations

### 17.1 Migration Rules

- Every migration file is immutable once merged — never edit an applied migration
- Migration filenames must be timestamp-prefixed: `20250615_143000_add_index_users_email`
- Each migration must have a corresponding rollback (down) script
- Migrations run automatically during deployment — never applied manually
- Test every migration against a copy of production data before deploying

### 17.2 Zero-Downtime Migration — Expand-Contract Pattern

```
// Safe zero-downtime steps for renaming a column

// Phase 1 — Expand
MIGRATION: ADD COLUMN new_name VARCHAR(255) NULLABLE
CODE: write to both old_name AND new_name; read from old_name

// Phase 2 — Migrate data
MIGRATION: UPDATE table SET new_name = old_name WHERE new_name IS NULL

// Phase 3 — Switch reads
CODE: write to both; read from new_name

// Phase 4 — Contract
CODE: write to new_name only
MIGRATION: DROP COLUMN old_name

// ❌ NEVER: rename or drop a column in a single migration while app is running
```

### 17.3 Destructive Migration Policy

- Dropping a column or table requires a 2-week notice in the PR
- Never truncate a production table in a migration — use a background job for bulk deletes
- Adding a NOT NULL column to an existing table requires a default value or multi-phase migration
- Seed data is managed by dedicated seed scripts, not migration files

---

## 18. Async Jobs, Queues & Events

### 18.1 When to Use a Queue

- Email and notification delivery
- PDF generation, report generation, data export
- Third-party API calls not required for the immediate response
- Any operation that takes more than 2 seconds
- Webhook delivery and retry
- Scheduled recurring jobs

### 18.2 Job Design Rules

- Every job must be **idempotent** — running the same job twice must produce the same result
- Job payloads contain only IDs and minimal context — never large nested objects
- Maximum 3 automatic retries with exponential backoff before dead-letter queue
- Dead-letter queue items must trigger an alert and be reviewed within 24 hours

```
JOB send-welcome-email:
  payload: { userId: 'usr_abc123' }  // ID only
  maxRetries: 3
  backoff:    exponential (1s, 2s, 4s)
  timeout:    30 seconds

JOB HANDLER:
  user = userRepo.findById(payload.userId)
  IF NOT user: DISCARD  // user deleted — safe to skip

  // Idempotency check
  IF user.welcomeEmailSentAt IS NOT NULL: RETURN

  emailService.sendWelcome(user)
  userRepo.update(user.id, { welcomeEmailSentAt: NOW() })
```

### 18.3 Event Publishing Standards

- Domain events are published **after** a successful commit — never before
- Event names use past tense: `UserCreated`, `OrderShipped`, `PaymentFailed`
- Event payloads follow the same DTO standards as API responses
- Events are versioned: `UserCreated.v1`, `UserCreated.v2`
- Consumers must handle out-of-order and duplicate events (idempotent consumers)

---

## 19. Concurrency & Consistency

### 19.1 Optimistic Locking

```
MODEL Order:
  version: INTEGER  NOT NULL  DEFAULT 1

FUNCTION updateOrder(id, data, expectedVersion):
  rowsAffected = db.update('orders',
    SET   = data,
    WHERE = { id: id, version: expectedVersion }
    INCREMENT version = version + 1
  )
  IF rowsAffected == 0:
    THROW ConflictError('Order was modified by another request. Please reload and try again.')
```

### 19.2 Idempotency Keys

```
ENDPOINT POST /payments:
  idempotencyKey = request.headers['Idempotency-Key']
  IF NOT idempotencyKey: THROW BadRequestError('Idempotency-Key header required')

  existing = idempotencyRepo.find(idempotencyKey)
  IF existing:
    RETURN existing.response  // return stored response, no double charge

  result = paymentService.charge(request.body)
  idempotencyRepo.save({ key: idempotencyKey, response: result, expiresAt: NOW + 24h })
  RETURN result
```

> ❌ Payment, order creation, and wallet debit endpoints MUST implement idempotency keys. This is a release blocker.

---

## 20. Data Privacy & Retention

### 20.1 PII Classification

| Class | Examples | Rules |
|---|---|---|
| Sensitive PII | Password, SSN, financial data, health data | Encrypt at rest. Never log. Never include in response DTOs. |
| Personal PII | Name, email, phone, IP address, device ID | Mask in logs. Exclude from analytics exports. Honour deletion requests. |
| Quasi-identifier | Age range, region, job title | Anonymise before analytics use. |
| Non-personal | Product names, categories, public prices | Standard handling. |

### 20.2 Data Handling Rules

- Mask PII in all logs: `a*****@example.com`, `***-***-1234`
- Audit trail: log who accessed or modified PII records and when
- Delete or anonymise user data within 30 days of account deletion request
- Hard delete must be executed for GDPR/deletion requests (soft delete is not sufficient)
- Never include PII in URLs, query parameters, or error messages
- Encrypt sensitive PII fields at rest using AES-256 or equivalent

```
FUNCTION maskForLog(user):
  RETURN {
    id:    user.id,
    email: user.email[0] + '*****@' + user.email.split('@')[1],
    phone: '***-***-' + user.phone.slice(-4),
    name:  user.name[0] + '***'
  }

// Usage
logger.info('User logged in', { user: maskForLog(user) })
```

---

## 21. Naming Conventions

| Construct | Convention | Example |
|---|---|---|
| Source files | kebab-case | `user-auth.service` / `order.repository` |
| Classes | PascalCase | `UserService` / `OrderRepository` |
| Functions & Methods | camelCase | `getUserById()` / `calculateDiscount()` |
| Variables | camelCase | `accessToken` / `currentUser` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_LIMIT` / `JWT_EXPIRY` |
| Interfaces | PascalCase + I prefix | `IUserRepository` / `IPaymentProvider` |
| Enums | PascalCase | `OrderStatus` / `HttpStatusCode` |
| Database tables | snake_case plural | `order_items` / `user_roles` |
| Database columns | snake_case | `created_at` / `deleted_at` / `user_id` |
| Environment variables | UPPER_SNAKE_CASE | `DATABASE_URL` / `JWT_PRIVATE_KEY` |
| REST endpoints | kebab-case plural | `/user-profiles` / `/order-items/:id` |
| DTO classes | PascalCase + suffix | `CreateUserDTO` / `UserResponseDTO` |
| Test files | Same name + .test | `user.service.test` / `UserCard.test` |
| Migration files | timestamp + description | `20250615_143000_add_users_email_index` |
| Queue job names | kebab-case past tense | `send-welcome-email` / `process-payment` |
| Event names | PascalCase past tense | `UserCreated` / `OrderShipped` |

---

## 22. Code Review Checklist

Every pull request must pass all blocker items before approval.

| # | Checklist Item | Blocker? |
|---|---|---|
| 1 | Layer boundaries respected — no repo calls in controllers, no HTTP objects in services | ✅ Yes |
| 2 | Each class has a single, clearly defined responsibility | ✅ Yes |
| 3 | Dependencies are injected — no `new ConcreteClass()` inside services or controllers | ✅ Yes |
| 4 | All DB queries are inside repository methods only | ✅ Yes |
| 5 | No N+1 queries — list endpoints use eager loading or batch loading | ✅ Yes |
| 6 | All model relations declared explicitly on both sides with aliases and foreign keys | ✅ Yes |
| 7 | DTOs used at all layer boundaries — raw ORM models never returned to clients | ✅ Yes |
| 8 | Request DTOs validated in middleware before reaching the service layer | ✅ Yes |
| 9 | Validation errors are field-specific, human-readable, all returned in one response | ✅ Yes |
| 10 | Sensitive fields excluded from all response DTOs | ✅ Yes |
| 11 | Multi-table writes wrapped in a database transaction | ✅ Yes |
| 12 | Unknown/extra request fields stripped before processing | ✅ Yes |
| 13 | Business-rule violations return 409, not 422 | No |
| 14 | All endpoints require authentication unless explicitly marked public | ✅ Yes |
| 15 | Ownership checks applied for user-scoped resources | ✅ Yes |
| 16 | Passwords hashed with bcrypt/argon2 — never plain or MD5/SHA | ✅ Yes |
| 17 | No secrets or credentials hardcoded — all from environment variables | ✅ Yes |
| 18 | All queries use parameterised inputs — no string concatenation in SQL | ✅ Yes |
| 19 | Response envelope follows the standard shape (success, statusCode, data, traceId) | ✅ Yes |
| 20 | List endpoints support page/limit/sortBy/sortOrder query params | ✅ Yes |
| 21 | Structured logs include traceId and do not log PII or secrets | ✅ Yes |
| 22 | Cached data has an explicit TTL and a documented invalidation strategy | No |
| 23 | Migration is zero-downtime safe — no destructive changes while app is live | ✅ Yes |
| 24 | Background jobs are idempotent and have retry + dead-letter handling | ✅ Yes |
| 25 | Payment/order creation endpoints implement idempotency key handling | ✅ Yes |
| 26 | Unit tests mock all external dependencies | ✅ Yes |
| 27 | Each test follows Arrange → Act → Assert structure | No |
| 28 | Test names describe expected behaviour in plain language | No |
| 29 | No test depends on another test or on execution order | ✅ Yes |
| 30 | Service coverage ≥ 90%, overall project ≥ 80% | ✅ Yes |

---

_Engineering Team · Backend API Development Standards v1.2 · 2025 · Internal Use Only_
