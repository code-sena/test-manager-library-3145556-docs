# Architecture — Manager Library (Python)

**Hexagonal** architecture (ports and adapters). The dependency rules are Articles I to III of
[`constitution.md`](constitution.md).

## 1. Layers and dependencies

```mermaid
flowchart LR
    subgraph Inbound
      HTTP["adapters/inbound/http<br/>(FastAPI)"]
    end
    subgraph Core
      APP["application<br/>use cases + ports"]
      DOM["domain<br/>aggregates and value objects"]
    end
    subgraph Outbound
      DB["adapters/outbound/persistence<br/>(SQLAlchemy 2 + Alembic)"]
      STO["adapters/outbound/storage<br/>covers"]
      SEC["adapters/outbound/security<br/>hashing and token"]
    end
    HTTP --> APP --> DOM
    DB -. implements ports of .-> APP
    STO -. implements .-> APP
    SEC -. implements .-> APP
```

Dependency arrows point **inward**. `bootstrap` is the only thing that sees both halves.

## 2. Folder structure

```text
src/library/
├── domain/              # Book, Loan, LoanItem, Member, User; Isbn, Quantity, DateRange
├── application/
│   ├── ports/outbound/  # BookRepository, LoanRepository, MemberRepository, UserRepository,
│   │                    # ReportReader, Clock, PasswordHasher, TokenIssuer, CoverStorage
│   └── services/        # LendBooksService, ReturnItemService, CreateBookService, SearchBooksService,
│                        # OverdueService, MostBorrowedReportService, LoginService, CreateLibrarianService
├── adapters/
│   ├── inbound/http/    # routers/controllers, request schemas, error handling
│   └── outbound/        # persistence/, storage/, security/
└── bootstrap/composition/   # src/library/bootstrap/composition/port_bindings.py — the single composition point
```

## 3. Aggregates

| Root | Contains | Invariants it defends |
|---|---|---|
| `Book` | `Isbn`, total and available copies, soft delete | available between 0 and total; total ≥ 1; non-empty title |
| `Loan` | a collection of `LoanItem` | 1 to 3 lines; no repeated book; 14-day due date; immutable except for returns |
| `Member` | document number | document number required and unique |
| `User` | role, hash | normalized username; role in (`admin`, `librarian`) |

`Loan.AddItem` calls `Book.Withdraw` before adding the line: withdrawing the copy and creating the
line are **a single operation**.
`Loan.ReturnItem` calls `Book.Restock` exactly once per line.

## 4. Flow: registering a loan

```mermaid
sequenceDiagram
    participant C as Client
    participant H as HTTP
    participant S as LendBooksService
    participant M as MemberRepository
    participant B as BookRepository
    participant L as LoanRepository
    C->>H: POST /api/v1/loans
    H->>S: Lend(memberId, bookIds)
    S->>M: fetch member + unreturned lines
    S->>S: any overdue? more than 3?
    S->>B: fetch books in batch (Q3)
    S->>S: Loan.AddItem → Book.Withdraw (for each book)
    S->>L: save loan + books (one transaction)
    L-->>S: ok or version conflict
    S-->>H: loan or BOOK_NOT_AVAILABLE
    H-->>C: 201 or 409
```

## 5. Errors

The domain and the application raise their own errors (`BookNotAvailable`, `MemberHasOverdue`,
`MemberLimitExceeded`, …). **Only** the HTTP adapter translates them into the single error body and
status code from [`api-contract.md`](api-contract.md). A framework error never reaches the client.

## 6. What there is not

No events, no queues, no cache, no microservices, no multi-tenancy. These get added if a requirement
asks for them, not before.
