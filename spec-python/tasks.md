# Tasks — Manager Library (Python)

Each task is done when its **evidence** exists and can be run (Article XII). Initial state of all:
**pending**.

| # | Task | Depends on | Evidence |
|---|---|---|---|
| T-01 | Project skeleton and `GET /health` | — | `GET /health` responds `{"status":"ok"}`; the container boots with a single command |
| T-02 | Initial migration: six tables, FKs, unique indexes and the five seeded categories | T-01 | The queries from `data-model.md` §10 return what is documented; no column has a `DEFAULT` |
| T-03 | Domain: `Isbn`, `Quantity`, `DateRange` (value objects) | T-01 | Unit tests: 13-digit ISBN, quantity > 0, end ≥ start |
| T-04 | Domain: `Book` (`Withdraw`, `Restock`, soft delete) | T-03 | Tests: never < 0, never > total |
| T-05 | Domain: `Loan` (`AddItem`, `ReturnItem`, computed status, 14 days, max 3) | T-04 | Tests with a fake `Clock`: `open`, `overdue`, `closed` |
| T-06 | Domain: `Member` and `User` (name normalization, valid role) | T-03 | `"Ana "` becomes `ana`; the `admin` role is rejected through the API |
| T-07 | **Authentication** ports and use cases (`LoginService`, `CreateLibrarianService`) with fakes | T-06 | US-01 and US-02 pass against fakes |
| T-08 | **Books** ports and use cases (create, search, edit, soft delete, cover) with fakes | T-04 | US-03 and US-04 pass against fakes |
| T-09 | **Lend** use case (`LendBooksService`) with fakes | T-05, T-08 | US-06 complete: limits, overdue, no copies left |
| T-10 | **Return** and **overdue** use cases | T-09 | US-07 and US-09; returning twice → `ITEM_ALREADY_RETURNED` |
| T-11 | **Most-borrowed report** use case and its read port | T-09 | US-10; the old title is kept after a rename (ADR-004) |
| T-12 | Persistence adapter: book, member, user repositories | T-02, T-08 | Integration tests against a real PostgreSQL |
| T-13 | Persistence adapter: loan repository + version witness | T-12, T-09 | Two loans of the last copy: one 201, the other 409 (NFR-2) |
| T-14 | Security adapter: password hashing and JWT token | T-07 | The password never appears in logs or responses (NFR-3) |
| T-15 | Cover adapter (disk) | T-08 | Uploading a 2 MB PNG: ok; 3 MB: 422; no cover: `cover_key` null |
| T-16 | HTTP adapter: routes from [`api-contract.md`](api-contract.md), single error body, role-based authorization | T-10, T-11, T-14, T-15 | A contract test walks every route and every code in §2 |
| T-17 | Composition and bootstrap (initial admin from the environment) | T-16 | The system boots with an empty database and an admin can log in |
| T-18 | Push the **domain-only** invariants down to the database: `ck_book_total_positive`, `ck_user_role` | T-12 | An `INSERT` through `psql` that violates them fails; `data-model.md` §4 gets updated |
| T-19 | Architecture tests (Articles I, II and III) | T-17 | An `import-linter` contract and a test in `tests/architecture/` pass, and fail if a forbidden import is introduced |
| T-20 | `infra`: `docker compose up` boots an empty engine, the API and the frontend | T-17 | One command; the database boots with no tables until the migrations run |
| T-21 | `tool`: demo seeder **through the API** (≥ 40 books, 10 members, 15 loans, 3 overdue) | T-17 | After running it, `/loans/overdue` returns 3 lines and the report has rows |
| T-22 | `app`: login, catalog, members, lend, return, overdue, report | T-16 | A manual end-to-end walkthrough at the fair |
| T-23 | `page`: static public presentation site | — | Opens without the API; correct on mobile and desktop |

## Definition of done for the test

Whoever reaches T-17 with evidence has the **core** done. T-18 through T-23 are what separates good
work from outstanding work. What is evaluated is SDD performance, not the number of tasks: one task
with evidence is worth more than three half-done.
