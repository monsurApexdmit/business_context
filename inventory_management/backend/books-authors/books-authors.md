# Module: Books & Authors

## Business Purpose
A simple legacy CRUD module for managing books and authors. Not scoped by company_id. This module appears to be a starter/demo module. Books belong to authors. No soft delete on either table.

---

## Database Tables

### Table: `authors`
| Column     | Type          | Notes             |
|------------|---------------|-------------------|
| id         | uint (PK)     | json: id          |
| name       | varchar(255)  | json: name        |
| email      | varchar(255)  | json: email       |
| created_at | timestamp     | json: created_at  |
| updated_at | timestamp     | json: updated_at  |

### Table: `books`
| Column     | Type          | Notes                              |
|------------|---------------|------------------------------------|
| id         | uint (PK, AI) | json: id                           |
| title      | string        | json: title                        |
| author_id  | uint (FK)     | FK → authors.id; json: author_id   |
| created_at | timestamp     | json: created_at                   |
| updated_at | timestamp     | json: updated_at                   |

---

## Relationships
- Book → Author (BelongsTo via author_id)

---

## Business Logic
- No company scoping.
- No soft delete — hard delete on both tables.
- Delete returns `204 No Content`.
- On create/update book: validates that `author_id` references an existing author.

---

## Endpoints — Authors

### GET /authors/
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Authors retrieved successfully",
  "data": [
    { "id": 1, "name": "George Orwell", "email": "orwell@example.com", "created_at": "...", "updated_at": "..." }
  ]
}
```

---

### GET /authors/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Author fetched successfully",
  "data": { "id": 1, "name": "George Orwell", "email": "orwell@example.com", "created_at": "...", "updated_at": "..." }
}
```

**Error:** `404` — `{"error": "Author not found"}`

---

### POST /authors/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{ "name": "George Orwell", "email": "orwell@example.com" }
```

**Response 201:**
```json
{
  "message": "Author created successfully",
  "data": { /* author object */ }
}
```

---

### PUT /authors/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:** Author fields to update.

**Response 200:**
```json
{
  "message": "Author updated successfully",
  "data": { /* author object */ }
}
```

---

### DELETE /authors/:id
**Auth:** Bearer JWT required

**Response 204 No Content**

---

## Endpoints — Books

### GET /books/
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Books retrieved successfully",
  "data": [
    {
      "id": 1,
      "title": "1984",
      "author_id": 1,
      "author": { "id": 1, "name": "George Orwell", "email": "orwell@example.com" },
      "created_at": "...",
      "updated_at": "..."
    }
  ]
}
```

---

### GET /books/:id
**Auth:** Bearer JWT required

**Response 200:**
```json
{
  "message": "Book fetched successfully",
  "data": { /* book with author preloaded */ }
}
```

**Error:** `404` — `{"error": "Book not found"}`

---

### POST /books/
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{ "title": "1984", "author_id": 1 }
```

**Validation:**
- `author_id` — must reference an existing author

**Response 201:**
```json
{
  "message": "Book created successfully",
  "data": { /* book with author preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid JSON body"}` / `{"error": "Author not found"}`
- `500` — `{"error": "Failed to create book"}` / `{"error": "Failed to load created book"}`

---

### PUT /books/:id
**Auth:** Bearer JWT required
**Content-Type:** `application/json`

**Request Body:**
```json
{ "title": "Animal Farm", "author_id": 1 }
```

**Validation:** `author_id` — must reference an existing author

**Response 200:**
```json
{
  "message": "Book updated successfully",
  "data": { /* book with author preloaded */ }
}
```

**Error Responses:**
- `400` — `{"error": "Invalid JSON body"}` / `{"error": "Author not found"}`
- `404` — `{"error": "Book not found"}`
- `500` — `{"error": "Failed to update book"}`

---

### DELETE /books/:id
**Auth:** Bearer JWT required

**Response 204 No Content**

**Error:** `500` — `{"error": "Failed to delete book"}`

---

## Dependencies
- None (standalone demo module)
