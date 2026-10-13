# Technical plan — Manager Library (Python)

How what [`spec.md`](spec.md) and [`constitution.md`](constitution.md) describe gets built. The
schema is **not** described here: it lives in [`data-model.md`](data-model.md). The shape of the
system, in [`architecture.md`](architecture.md).

## 1. Technical decisions

| # | Decision | Choice (Python) | Why |
|---|---|---|---|
| D-01 | Language and runtime | Python 3.12 | Stack of the reference spec |
| D-02 | HTTP framework | FastAPI | Contract in [`api-contract.md`](api-contract.md) |
| D-03 | Persistence | SQLAlchemy 2 + Alembic over PostgreSQL 16 | The schema belongs to the migrations (ADR-001) |
| D-04 | Concurrency | Per-book version witness + range `CHECK` ([ADR-002](adr/adr-002-optimistic-concurrency.md)) | Last copy (NFR-2) |
| D-05 | Dates | UTC instants (`timestamptz`); the clock is a port (`Clock`) | Test due dates without waiting 14 days |
| D-06 | Report | One aggregated query behind a read port ([ADR-004](adr/adr-004-aggregated-frozen-report.md)) | Stable and cheap |
| D-07 | Value objects | `Isbn`, `Quantity`, `DocumentNumber`, `DateRange`: live inside their owner's row, no table | They have no identity |
| D-08 | Covers | Binary in external storage (disk locally, S3 in production); the database only stores `cover_key` | Opaque key, never a path |
| D-09 | Passwords | Argon2id behind the `PasswordHasher` port; the domain never sees the plain password | Article IX |
| D-10 | Bootstrap | The application creates the initial admin from `ADMIN_USERNAME` / `ADMIN_PASSWORD` | The database cannot produce the hash |
| D-11 | Authentication | JWT HS256, 8 hours, key in `JWT_SECRET` | Simple and stateless |
| D-12 | Tests | pytest, with fakes for the ports and a real database only for integration tests | Test pyramid |

## 2. Per-environment configuration

| Variable | Required | Example |
|---|---|---|
| `DATABASE_URL` | yes | `postgresql://library:library@db:5432/library` |
| `JWT_SECRET` | yes | a long random string |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | yes | defined in `.env`, never in the repo |
| `COVERS_DIR` | no | `./var/covers` |
| `CORS_ORIGIN` | yes | `http://localhost:5173` |

## 3. Test strategy

| Level | What it tests | Against what |
|---|---|---|
| Domain | Invariants: maximum 3, 14-day due date, available copies in range | Nothing: pure objects |
| Application | Use cases: lend, return, report | Port fakes |
| Integration | Repositories and the report query | Real PostgreSQL |
| Contract | Single error body and status codes | The booted API |
| Architecture | Articles I, II and III | an `import-linter` contract and a test in `tests/architecture/` |
| Concurrency | Two loans of the last copy | Real PostgreSQL |

## 4. Build order

Skeleton and health → initial migration with seed → domain → use cases with fakes → adapters → API →
contract and concurrency tests.
The detail, with evidence, is in [`tasks.md`](tasks.md).
