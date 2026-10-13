# API contract — Manager Library

The HTTP contract is **language-independent**: whoever implements it in React + Laravel must respect
it exactly as whoever implements it in Python or .NET. What each operation does is in
[`spec.md`](spec.md); what is stored, in [`data-model.md`](data-model.md).

## 1. Conventions

| Topic | Rule |
|---|---|
| Base | `/api/v1` |
| Format | JSON, UTF-8. Field names in `snake_case` |
| Identifiers | UUID v4 as text |
| Dates | ISO 8601 in UTC, with `Z` (`2026-10-14T14:15:00Z`) |
| Money | None. The system handles no amounts |
| Authentication | `Authorization: Bearer <token>` on everything except `POST /auth/login` and `GET /health` |
| Pagination | `?page=1&size=20`. `size` max 50. The response is `{ "items": [...], "page": 1, "size": 20, "total": 134 }` |
| Language | Error messages in Spanish; fields and codes in English |

### Error body (the same for every error)

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` may be empty. It **never** includes stack traces, SQL, server paths or the password.

### Status codes

| Status | When |
|---|---|
| 200 / 201 / 204 | Success / created / no content |
| 401 | No token, invalid or expired token, invalid credentials |
| 403 | Insufficient role |
| 404 | Does not exist (or has been soft-deleted) |
| 409 | State conflict: duplicate, no copies left, already returned, member with overdue items, book with open loans |
| 422 | Validation: invalid field, broken input rule, inverted date range |

## 2. Business error codes

| `code` | Status | Origin |
|---|---|---|
| `INVALID_CREDENTIALS` | 401 | AC-01.2 |
| `FORBIDDEN_ROLE` | 403 | AC-01.3 |
| `NOT_FOUND` | 404 | Any resource |
| `USERNAME_TAKEN` | 409 | AC-02.2 |
| `ISBN_TAKEN` | 409 | AC-03.2 |
| `DOCUMENT_TAKEN` | 409 | AC-05.1 |
| `BOOK_HAS_OPEN_LOANS` | 409 | AC-03.5 |
| `BOOK_NOT_AVAILABLE` | 409 | AC-06.3 |
| `MEMBER_HAS_OVERDUE` | 409 | AC-06.5 |
| `ITEM_ALREADY_RETURNED` | 409 | AC-07.2 |
| `MEMBER_LIMIT_EXCEEDED` | 422 | AC-06.4 |
| `DUPLICATE_BOOK_IN_LOAN` | 422 | AC-06.2 |
| `VALIDATION_ERROR` | 422 | Any invalid field |
| `INVALID_DATE_RANGE` | 422 | AC-10.2 |

## 3. Endpoints

### 3.1 Authentication and health

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/auth/login` | public | Returns a token |
| GET | `/health` | public | `{ "status": "ok" }` |

`POST /auth/login`

```json
// request
{ "username": "ana", "password": "********" }
// 200
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 28800, "role": "librarian" }
```

### 3.2 Users

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/users` | admin | Creates a librarian |

```json
// request  (role only accepts "librarian")
{ "username": "Ana ", "password": "********", "role": "librarian" }
// 201  (no password, no password_hash)
{ "id": "…", "username": "ana", "role": "librarian" }
```

### 3.3 Categories

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/categories` | authenticated | The five categories, ordered by name |

### 3.4 Books

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/books` | authenticated | Search: `q`, `category_id`, `page`, `size` |
| GET | `/books/{id}` | authenticated | Detail |
| POST | `/books` | admin | Creates |
| PUT | `/books/{id}` | admin | Replaces title, author, category and total |
| DELETE | `/books/{id}` | admin | Soft delete |
| PUT | `/books/{id}/cover` | admin | Uploads a cover (`multipart/form-data`, field `file`) |
| DELETE | `/books/{id}/cover` | admin | Removes the cover |

Book (response):

```json
{
  "id": "…", "title": "Cien años de soledad", "author": "Gabriel García Márquez",
  "isbn": "9780307474728", "category": { "id": "…", "name": "Novela" },
  "total_copies": 4, "available_copies": 3, "cover_key": null
}
```

`POST /books` (request): `title`, `author`, `isbn`, `category_id`, `total_copies`. Available copies
are computed by the server.
`cover_key` is an opaque key: to render the cover, use `GET /books/{id}/cover`, which returns the
binary or 404.

### 3.5 Members

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/members` | admin, librarian | List and search: `q`, `page`, `size` |
| GET | `/members/{id}` | admin, librarian | Detail with `active_loans` (count of unreturned items) |
| POST | `/members` | admin, librarian | Creates: `full_name`, `document_number` |

### 3.6 Loans

| Method | Route | Role | Description |
|---|---|---|---|
| POST | `/loans` | admin, librarian | Registers a loan |
| GET | `/loans` | admin, librarian | Lists: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | `/loans/{id}` | admin, librarian | Detail with lines and status |
| POST | `/loans/{loan_id}/items/{item_id}/return` | admin, librarian | Returns one line |
| GET | `/loans/overdue` | admin, librarian | Overdue lines |

`POST /loans`

```json
// request  (1 to 3 items, no repeated books)
{ "member_id": "…", "items": [ { "book_id": "…" }, { "book_id": "…" } ] }
// 201
{
  "id": "…", "member_id": "…", "loaned_at": "2026-10-14T14:15:00Z", "status": "open",
  "items": [
    { "id": "…", "book_id": "…", "book_title": "Cien años de soledad", "isbn": "9780307474728",
      "category_name": "Novela", "due_at": "2026-10-28T14:15:00Z", "returned_at": null }
  ]
}
```

`status` is computed: `open` (there are unreturned lines and none overdue), `overdue` (some
unreturned and overdue), `closed` (every line returned).

`POST /loans/{loan_id}/items/{item_id}/return` → 200 with the line (`returned_at` filled in).

`GET /loans/overdue` → `items`: `{ "loan_id", "item_id", "book_title", "member_name", "due_at",
"days_late" }`. Here the member's name **is** shown, because it is an operational listing; the
§3.7 report does **not** show it.

### 3.7 Report

| Method | Route | Role | Description |
|---|---|---|---|
| GET | `/reports/most-borrowed` | admin | `from` and `to` required (date, both inclusive) |

```json
{
  "from": "2026-10-01", "to": "2026-10-31",
  "rows": [
    { "book_id": "…", "book_title": "Cien años de soledad", "category_name": "Novela", "times_borrowed": 12 }
  ]
}
```

One row per frozen `book_id + book_title + category_name`. Order: `times_borrowed` descending, then
title. No members, no librarians.

## 4. Security

| Aspect | Rule |
|---|---|
| Token | Signed JWT, 8 hours, with `sub` and `role`. No personal data inside |
| Passwords | Only the hash is stored; the hash never leaves in any response |
| Authorization | Evaluated on every endpoint by role; a 403 does not reveal whether the resource exists |
| CORS | Only the frontend origin configured through the environment |
| Cover upload | Only `image/jpeg` and `image/png`, max 2 MB; the type is validated by content, not by extension |
