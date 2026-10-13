# ADR-001 · The schema is owned by the application's migrations

- **Status:** accepted
- **Context:** the `infra` repository boots a database engine. Someone could put DDL in a container
  bootstrap script and more of it in the application's migrations; with two owners, nobody knows
  which schema is actually running in production.
- **Decision:** **all DDL lives in the application's migrations** (tables, indexes, constraints and
  the category seed). `infra` only creates the empty engine, the user and the volume.
- **Alternatives discarded:** SQL scripts in `infra/db/init/` (a second source of truth); an ORM in
  "create tables on boot" mode (leaves no history and no rollback).
- **Consequences:** a new Postgres extension or index is created in the **same** migration that uses
  it. A schema changed by hand on the engine is a defect, not a variant.
- **How it is verified:** `infra` contains no `CREATE TABLE`; the queries in `data-model.md` §10
  match the migrations.
