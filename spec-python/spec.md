# Specification — Manager Library

**What is built and why.** A system for a small library to track its catalog of books, its members
and its loans, and to see which books are borrowed the most.

Governed by [`constitution.md`](constitution.md). The data model lives only in
[`data-model.md`](data-model.md) and the HTTP contract only in [`api-contract.md`](api-contract.md).

---

## 1. Actors

| Actor | What they do | How they authenticate |
|---|---|---|
| **Admin** (`admin`) | Maintains the book catalog and the members; onboards librarians; sees every report | Username and password → token |
| **Librarian** (`librarian`) | Registers loans and returns; looks up the catalog, members and overdue items | Username and password → token |
| **Member** | A person who borrows books. **Never logs in**: they are data, not a user | — |

The `admin` role is provisioned by the deployment from environment variables. **Nobody grants it at
runtime.**

## 2. Scope

**In:** book catalog with an optional cover, five fixed categories, members, loans with several
lines, per-line returns, overdue books, most-borrowed report, login with roles.

**Out, and not reopened:** fines and payments, reservations, renewals, email or SMS, bulk import,
inter-branch loans, book attributes beyond §3 (no publisher, no year, no language).

## 3. User stories and acceptance criteria

### US-01 · Log in
As a user I want to authenticate so I can operate according to my role.
- **AC-01.1** With valid credentials a token is issued, valid for 8 hours.
- **AC-01.2** With invalid credentials it responds 401 with a message that does **not** say whether
  the username or the password failed.
- **AC-01.3** Every endpoint except login and `/health` requires a token. No token: 401.
  Insufficient role: 403.

### US-02 · Onboard librarians (admin)
- **AC-02.1** The admin creates a user with a username, a password and the `librarian` role.
- **AC-02.2** The username is stored lowercased and trimmed; it is unique. Duplicate: 409.
- **AC-02.3** Requesting the `admin` role through the API responds 422. The password is never
  returned nor logged.

### US-03 · Manage books (admin)
- **AC-03.1** Create a book with title, author, ISBN, category and total copies. Available copies
  start equal to the total.
- **AC-03.2** The ISBN has 13 digits and is unique among active books. Duplicate: 409.
- **AC-03.3** Total copies ≥ 1. Changing the total adjusts the available copies by the same
  difference and **never** leaves available copies < 0 (422).
- **AC-03.4** Uploading a cover (JPG/PNG, max 2 MB) stores an **opaque key**; with no cover the field
  is `null`, never an empty string.
- **AC-03.5** Deleting a book is a **soft delete**: it stops appearing in the catalog but stays in
  past loans. A book with open loans is not deleted (409).

### US-04 · Search books
- **AC-04.1** Partial-text search on title or author, filter by category, active books only,
  ordered by title, paginated (`page`, `size`, max 50).
- **AC-04.2** The response includes the total result count.
- **AC-04.3** Looking up a book by a nonexistent or deleted id: 404.

### US-05 · Manage members
- **AC-05.1** The admin and the librarian create members with a name and a document number. The
  document number is unique: duplicate 409.
- **AC-05.2** Members are listed and searched by name or document number, paginated.
- **AC-05.3** Members are never deleted.

### US-06 · Register a loan
- **AC-06.1** A loan belongs to **one** member, is registered by the authenticated user, and carries
  1 to 3 lines.
- **AC-06.2** The same book cannot repeat within a loan (422).
- **AC-06.3** Each line withdraws **1** available copy in the same operation. With no copies left:
  409 and **nothing is recorded**.
- **AC-06.4** A member cannot have more than **3 unreturned books** in total (422 with the current
  count).
- **AC-06.5** A member with any **overdue** book cannot borrow another (409).
- **AC-06.6** The due date is `loaned_at + 14 days`. The line freezes the title, ISBN and category
  name **at that instant**.
- **AC-06.7** A registered loan is neither edited nor deleted.

### US-07 · Register a return
- **AC-07.1** **One line** is returned at a time. It sets `returned_at` and adds 1 to the available
  copies.
- **AC-07.2** Returning an already-returned line: 409. It **never** adds twice.
- **AC-07.3** Available copies never exceed the total (the database guarantees it).
- **AC-07.4** When every line is returned, the loan shows as **closed**.

### US-08 · View a loan and list loans
- **AC-08.1** Fetching a loan returns its lines and its computed status (`open`, `overdue`,
  `closed`).
- **AC-08.2** Listing by date range, status and member, ordered by date descending, paginated.
- **AC-08.3** The status **is computed**, not stored.

### US-09 · Overdue books
- **AC-09.1** Lists unreturned lines with `due_at` earlier than now: book, member, days late.
- **AC-09.2** Ordered by most days late first. `admin` and `librarian` only.

### US-10 · Most-borrowed books report
- **AC-10.1** Given a date range, one row per **book and frozen category name**, with the number of
  times borrowed, ordered highest to lowest.
- **AC-10.2** End before start: 422. A range with no loans: an empty list, not an error.
- **AC-10.3** The report is **not** broken down by librarian or by member.
- **AC-10.4** A report for a closed period **does not change** even if the book or its category is
  later renamed.
- **AC-10.5** `admin` only.

## 4. Business rules

| # | Rule |
|---|---|
| BR-1 | Available copies between 0 and total, always. |
| BR-2 | Loan period: 14 days. Maximum 3 unreturned books per member. |
| BR-3 | A member with overdue items cannot borrow more until they return them. |
| BR-4 | What is frozen in the line (title, ISBN, category) does not follow the catalog. |
| BR-5 | Loans and returns are never deleted. Books only get a soft delete. |
| BR-6 | Dates are UTC instants. User-facing messages in Spanish; code and columns in English. |

## 5. Non-functional requirements

| # | Requirement | How it is measured |
|---|---|---|
| NFR-1 | Book search responds in under 300 ms with 5,000 books | Load test with the `tool` seeder |
| NFR-2 | Two librarians lending the **last copy** at the same time: one wins, the other gets 409 | Concurrent test |
| NFR-3 | The password never travels or is stored in clear text; it never appears in logs | Review of logs and responses |
| NFR-4 | Every HTTP error uses the same body (see contract) | Contract test |
| NFR-5 | The system boots with a single command and the database starts empty | `infra` `README` |
