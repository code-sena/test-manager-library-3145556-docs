# Constitution — Manager Library (Python)

Non-negotiable principles of the project. They are not style or preference: each article carries
**how it is verified**, because a principle nobody can check is just an intention. A change that
violates an article is rejected without discussing the rest.

---

## Article I — The domain depends on nothing

`src/library/domain/` imports no other module of the project and no third-party library: only the
standard library (`dataclass(frozen=True)`, `datetime`, `typing`); no SQLAlchemy, no FastAPI, no
pydantic.

**Why:** it is the only way to test the invariants without a database, without a server and without
a container, and to change ORM without touching the rules.
**Verification:** an `import-linter` contract and a test in `tests/architecture/` forbid the domain
from depending on `application`, `adapters` or `bootstrap`.

## Article II — The core declares its ports; the world implements them

The application layer only knows the domain. Everything it needs from the outside (repositories,
clock, password hashing, cover storage, token generation) it expresses as a **port that it defines
itself**. No framework or ORM type appears in a port's signature.

**Verification:** an `import-linter` contract and a test in `tests/architecture/` prevent
`application` from importing `adapters`. Application tests run with fakes and never reference
`adapters`.

## Article III — A single composition point

A single module (`src/library/bootstrap/composition/port_bindings.py`) is where a port meets its
adapter. Outside of it, no infrastructure is instantiated.
**Verification:** no instantiation of an `adapters` class outside that module; checked by an
architecture test.

## Article IV — SOLID is not decorative

| Principle | Concrete application | Verification |
|---|---|---|
| **SRP** | One application service = one use case. `LendBooksService` lends and does nothing else | A service with two reasons to change is split |
| **OCP** | Switching cover storage (disk → S3) touches neither domain nor application | The new adapter only adds one line in the composition |
| **LSP** | Test fakes substitute the real adapter without changing expectations | The same test passes against the fake and the real one |
| **ISP** | Small ports: `Clock`, `PasswordHasher`, `TokenIssuer`, `CoverStorage` | No implementer leaves unused methods |
| **DIP** | The port is defined by whoever consumes it | Ports live in the application package |

## Article V — The schema is owned by the migrations

The `infra` repository boots an **empty** engine. All DDL (tables, indexes, constraints, extensions
and seed data) lives in the application's migrations (Alembic revisions). See
[ADR-001](adr/adr-001-schema-ownership.md).
**Verification:** `infra` contains no `CREATE TABLE`.

## Article VI — The invariants the database can enforce, the database enforces

A rule that only lives in code protects the application, not the data. Every invariant expressible in
the engine (`CHECK`, `UNIQUE`, FK) is declared there. What is not enforced is marked **pending** in
[`data-model.md`](data-model.md); it is not promised.
**Verification:** the queries in `data-model.md` §10 match what the document declares.

## Article VII — Derived state is computed, not stored

The state of a loan (`open`, `overdue`, `closed`) and the days late are computed. There is no column
for them.
**Verification:** `data-model.md` lists no `status` or `days_late` column.

## Article VIII — Frozen values do not follow the catalog

The loan line stores the title, ISBN and category of the book at the instant it was lent. No report
reads them from the live catalog.
**Verification:** a test that renames a book after it has been lent and checks that the report keeps
the old title ([ADR-004](adr/adr-004-aggregated-frozen-report.md)).

## Article IX — No secret enters the repository or a log

Not passwords, not precomputed hashes, not the token key. Every secret comes from the environment.
The real `.env` is in `.gitignore`; only `.env.example` is versioned.
**Verification:** `gitleaks` or a pattern `grep` in CI; no HTTP response includes `password` or
`password_hash`.

## Article X — The engine wins

If the data document and the database disagree, the document is broken and the document gets fixed.

## Article XI — Language

Code, columns and routes in English. Documentation and user-facing messages in Spanish.

## Article XII — Nothing is taken as done without evidence

A task in [`tasks.md`](tasks.md) is done when its **evidence** exists and can be run, not when "it's
already coded".
