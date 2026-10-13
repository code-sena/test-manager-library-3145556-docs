# Documentation Audit — Manager Library

**Repository:** `test-manager-library-3145556-docs`
**Audit date:** October 13, 2026
**Scope:** `spec-python/`, `spec-.net/`, and `README.md`
**Method:** read-only; no existing file was modified, created, or deleted other than this report.

---

## Finding types

| Type | Meaning |
|---|---|
| **Completeness** | Something is referenced but not defined, or only partially defined |
| **Consistency** | Two parts of the docs state the same thing in contradictory ways |
| **Bifurcation** | The spec offers two opposing paths or versions for the same shared rule |
| **Ambiguity** | A statement admits more than one valid interpretation |

**Severity:**
- **High** — a developer cannot implement correctly without resolving this point.
- **Medium** — raises doubt, but enough context exists to infer intent; worth clarifying.
- **Low** — minor detail that may not block implementation but clutters the documentation.

---

## Findings

### F-02
| Field | Value |
|---|---|
| **#** | 1 |
| **Type** | Completeness |
| **Severity** | High |
| **File** | `spec-python/data-model.md §3` and `spec-.net/data-model.md §3` (both specs) |

**Description:** ADR-002 and `plan.md D-04` require a version-witness column on `book` for optimistic concurrency. The physical model in `data-model.md §3` defines no such column (`version`, `row_version`, `xmin`, or any equivalent). A developer must invent the column name, type, and indexing strategy with no guidance from the authoritative data-model document.

**Exact quote — `spec-python/adr/adr-002-concurrencia-optimista.md`:**
> "El libro lleva un testigo de versión; al guardar, si la versión cambió, la operación falla…"
> (English: "The book carries a version witness; on save, if the version changed, the operation fails…")

**Exact quote — `spec-python/plan.md §1 D-04`:**
> "D-04 | Concurrencia | Testigo de versión por libro + `CHECK` de rango (ADR-002)"
> (English: "D-04 | Concurrency | Version witness per book + range `CHECK` (ADR-002)")

**`spec-python/data-model.md §3` (table `book`):** no column named `version` or equivalent is present in the table definition.

---

### F-04
| Field | Value |
|---|---|
| **#** | 2 |
| **Type** | Ambiguity |
| **Severity** | High |
| **File** | `spec-python/spec.md CA-06.3` and `spec-.net/spec.md CA-06.3` (both specs) |

**Description:** When a multi-book loan contains one or more books with no available copies, the spec says 409 and "nothing is recorded". It does not state whether the error body should report only the first exhausted book or all of them. The contract defines `details` as an array, which could suggest multiple entries, but no rule or example explicitly confirms this.

**Exact quote — `spec.md CA-06.3`:**
> "Cada línea descuenta **1** ejemplar disponible en la misma operación. Sin ejemplares: 409 y **no se registra nada**."
> (English: "Each line discounts **1** available copy in the same operation. No copies available: 409 and **nothing is recorded**.")

**Exact quote — `api-contract.md §1` error body example:**
```json
{ "code": "BOOK_NOT_AVAILABLE", "message": "…", "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ] }
```
The example shows a single entry in `details`. Whether this is exhaustive or only illustrative is not stated.

---

### F-03
| Field | Value |
|---|---|
| **#** | 3 |
| **Type** | Ambiguity |
| **Severity** | Medium |
| **File** | `spec-python/spec.md §5 RNF-1` and `spec-.net/spec.md §5 RNF-1` (both specs) |

**Description:** RNF-1 names the `tool` seeder as the measurement vehicle but does not specify the number of concurrent requests, whether the 5,000 books must all be active, or the target environment (local container, CI, etc.). The requirement is measurable in principle (300 ms with 5,000 books) but the test cannot be designed or reproduced without those parameters.

**Exact quote:**
> "RNF-1 | La búsqueda de libros responde en menos de 300 ms con 5 000 libros | Prueba de carga con el sembrador de `tool`"
> (English: "RNF-1 | Book search responds in under 300 ms with 5,000 books | Load test with the `tool` seeder")

---

### F-06
| Field | Value |
|---|---|
| **#** | 4 |
| **Type** | Completeness |
| **Severity** | Medium |
| **File** | `spec-python/api-contract.md §3.4` and `spec-.net/api-contract.md §3.4` (both specs) |

**Description:** `GET /books/{id}/cover` is mentioned in the explanatory prose but does not appear in the endpoint table of §3.4. A developer working only from the table will omit this endpoint. Additionally, the response content type (`Content-Type`) is not documented anywhere for this endpoint.

**Exact quote — prose in `api-contract.md §3.4`:**
> "`cover_key` es una clave opaca: para pintar la portada se usa `GET /books/{id}/cover`, que devuelve el binario o 404."
> (English: "`cover_key` is an opaque key: to render the cover use `GET /books/{id}/cover`, which returns the binary or 404.")

**Endpoint table in `api-contract.md §3.4`:** lists `PUT /books/{id}/cover` and `DELETE /books/{id}/cover` but not `GET /books/{id}/cover`.

---

### F-07
| Field | Value |
|---|---|
| **#** | 5 |
| **Type** | Completeness |
| **Severity** | Medium |
| **File** | `spec-python/api-contract.md §3.4` and `spec-.net/api-contract.md §3.4` (both specs) |

**Description:** `PUT /books/{id}` is described as "replaces title, author, category and total". The `isbn` field is not mentioned. Neither the contract nor `spec.md HU-03` states whether the ISBN is immutable after creation or whether it can be updated with `PUT`. Given that ISBN uniqueness among active books is enforced by a partial index (`data-model.md §4`), the omission of this rule is significant.

**Exact quote — `api-contract.md §3.4`:**
> "PUT | `/books/{id}` | admin | Reemplaza título, autor, categoría y total"
> (English: "PUT | `/books/{id}` | admin | Replaces title, author, category and total")

**Exact quote — `spec.md CA-03.2`:**
> "El ISBN tiene 13 dígitos y es único entre libros activos. Repetido: 409."
> (English: "The ISBN has 13 digits and is unique among active books. Duplicate: 409.")

CA-03.2 covers only creation. No acceptance criterion covers updating the ISBN.

---

### F-08
| Field | Value |
|---|---|
| **#** | 6 |
| **Type** | Ambiguity |
| **Severity** | Medium |
| **File** | `spec-python/api-contract.md §3.5` and `spec-.net/api-contract.md §3.5` (both specs) |

**Description:** `GET /members/{id}` returns a field called `active_loans` described as "conteo sin devolver" (count of unreturned items). It is not specified whether this count includes overdue lines (which are unreturned) or only open non-overdue lines. Since CA-06.5 blocks new loans when a member has any overdue item, this distinction affects what the frontend can infer about the member's loan status.

**Exact quote — `api-contract.md §3.5`:**
> "GET | `/members/{id}` | admin, librarian | Detalle con `active_loans` (conteo sin devolver)"
> (English: "GET | `/members/{id}` | admin, librarian | Detail with `active_loans` (count of unreturned items)")

---

### F-09
| Field | Value |
|---|---|
| **#** | 7 |
| **Type** | Ambiguity |
| **Severity** | Medium |
| **File** | `spec-python/spec.md CA-10.1` and `spec-.net/spec.md CA-10.1` (both specs) |

**Description:** The most-borrowed report filters by a date range (`from`/`to`), but no document specifies which column drives the filter. `loan.loaned_at` is the natural candidate, but `loan_item.due_at` or `loan_item.returned_at` could be equally argued. ADR-004 discusses grouping by frozen values but does not name the filter column.

**Exact quote — `spec.md CA-10.1`:**
> "Dado un rango de fechas, una fila por **libro y nombre de categoría congelado**, con la cantidad de veces prestado, ordenado de mayor a menor."
> (English: "Given a date range, one row per **book and frozen category name**, with the number of times borrowed, sorted highest to lowest.")

**Exact quote — `api-contract.md §3.7`:**
> "`from` y `to` obligatorios (fecha, ambos inclusive)"
> (English: "`from` and `to` required (date, both inclusive)")

Neither source names the column used to filter.

---

### F-10
| Field | Value |
|---|---|
| **#** | 8 |
| **Type** | Ambiguity |
| **Severity** | Medium |
| **File** | `spec-python/spec.md CA-03.2` and `spec-.net/spec.md CA-03.2` (both specs) |

**Description:** The ISBN is defined as "13 digits" but it is never stated whether the check digit (ISBN-13 modulo-10 checksum) must also be validated. A developer can implement strict validation (structure + checksum) or loose validation (length and digits only), and both readings are compatible with the documentation.

**Exact quote — `spec.md CA-03.2`:**
> "El ISBN tiene 13 dígitos y es único entre libros activos."
> (English: "The ISBN has 13 digits and is unique among active books.")

**Exact quote — `data-model.md §3` (column `isbn`):**
> "`isbn` | `char(13)` | no | Solo dígitos. Único entre libros activos (índice parcial)"
> (English: "`isbn` | `char(13)` | no | Digits only. Unique among active books (partial index)")

---

### F-11
| Field | Value |
|---|---|
| **#** | 9 |
| **Type** | Completeness |
| **Severity** | Medium |
| **File** | `spec-python/tasks.md` and `spec-.net/tasks.md` (both specs) |

**Description:** No task covers the application-layer service for managing members (HU-05: create, list, search). T-12 covers the persistence adapter for "libro, socio, usuario", but there is no task equivalent to T-07 (auth services with fakes), T-08 (book services with fakes), or T-09 (lending service with fakes) for the member use cases. The member service layer is absent from the task list.

**Relevant tasks from `tasks.md`:**

| Task | Covers |
|---|---|
| T-07 | Auth services with fakes → HU-01, HU-02 |
| T-08 | Book services with fakes → HU-03, HU-04 |
| T-09 | Lending service with fakes → HU-06 |
| *(missing)* | Member services with fakes → HU-05 |

---

### F-12
| Field | Value |
|---|---|
| **#** | 10 |
| **Type** | Completeness |
| **Severity** | Medium |
| **File** | `spec-python/plan.md §2` and `spec-.net/plan.md §2` (both specs) |

**Description:** `COVERS_DIR` is listed as optional. The table column is labelled "Ejemplo" (Example), not "Default", so `./var/covers` is only an example value, not a documented default. There is no statement of what happens when the variable is omitted: does the system use that path as a hardcoded fallback, fail on startup, or silently disable cover functionality?

**Exact quote — `plan.md §2`:**
> "`COVERS_DIR` | no | `./var/covers`"

The column header for the third column is "Ejemplo" (English: "Example"), not "Default".

---

### F-13
| Field | Value |
|---|---|
| **#** | 11 |
| **Type** | Consistency — possible inconsistency |
| **Severity** | Low |
| **File** | `spec-python/spec.md CA-06.4` vs. `api-contract.md §1` status codes (both specs) |

**Description:** CA-06.4 assigns HTTP 422 to "member exceeds 3 unreturned books", while CA-06.5 assigns 409 to "member has overdue items". Both rules block a new loan. The contract's own definitions state 409 = "state conflict" and 422 = "validation / broken input rule". Exceeding a count limit is a state conflict (like having overdue items), yet receives 422. The spec is internally consistent (CA-06.4 and the error code table agree on 422), but the semantic assignment is debatable and may confuse implementors.

**Exact quote — `spec.md CA-06.4`:**
> "Un socio no puede tener más de **3 libros sin devolver** en total (422 con el conteo actual)."
> (English: "A member cannot have more than **3 unreturned books** in total (422 with the current count).")

**Exact quote — `spec.md CA-06.5`:**
> "Un socio con algún libro **vencido** no puede pedir otro (409)."
> (English: "A member with any **overdue** book cannot request another (409).")

**Exact quote — `api-contract.md §1`:**
> "409 | Conflicto de estado: duplicado, sin ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos"
> (English: "409 | State conflict: duplicate, no copies, already returned, member with overdue, book with open loans")
>
> "422 | Validación: campo inválido, regla de entrada rota, rango de fechas invertido"
> (English: "422 | Validation: invalid field, broken input rule, inverted date range")

---

### F-15
| Field | Value |
|---|---|
| **#** | 12 |
| **Type** | Ambiguity |
| **Severity** | Low |
| **File** | `spec-python/spec.md CA-04.1` and `spec-.net/spec.md CA-04.1` (both specs) |

**Description:** "Partial text search on title or author" does not specify case sensitivity or Unicode normalization. An implementor can use case-sensitive prefix matching, case-insensitive substring (`ILIKE`), or full-text search (`to_tsvector`), and all are compatible with the wording.

**Exact quote — `spec.md CA-04.1`:**
> "Búsqueda por texto parcial en título o autor, filtro por categoría, solo libros activos, orden por título, paginada (`page`, `size`, máx. 50)."
> (English: "Partial text search on title or author, filter by category, active books only, ordered by title, paginated (`page`, `size`, max 50).")

---

## Findings considered and discarded

The following were examined and determined to be non-issues or outside the spec's responsibility:

| Candidate | Reason discarded |
|---|---|
| F-01 — `bootstrap/` placement in the folder tree | The tree unambiguously places `bootstrap/` as a child of `src/library/`. The inline comment is redundant but not misleading. Article III of `constitution.md` correctly names `src/library/bootstrap/composition/port_bindings.py` (Python) and `src/Library.Bootstrap/Composition/PortBindings.cs` (.NET). Format detail only; no implementation risk. |
| F-05 — `/health` prefix ambiguity | All endpoint tables in `api-contract.md` list routes without the `Base: /api/v1` prefix — `/books`, `/loans`, `/members`, `/auth/login`, and `/health` all follow the same convention. The prefix is uniformly implicit for every route. The `infra/README.md` example (`/api/v1/health`) is in a different repository and outside the spec's scope. No ambiguity within the spec itself. |
| Argon2id (Python) vs. BCrypt (.NET) in `plan.md D-09` | `constitution.md` and all ADRs define only the `PasswordHasher` port. The algorithm is a stack-specific technical decision, not a shared business rule. The difference is intentional and appropriate. |
| Asymmetry `member` has no `deleted_at` | CA-05.3 explicitly states "Los socios no se eliminan" ("Members are not deleted"). This directly explains the absence of a `deleted_at` column: no soft-delete is needed because deletion is ruled out entirely. The asymmetry with `book` is a design choice, not an omission. |
| `GET /loans/overdue` vs. `GET /loans/{id}` route conflict | This is a router implementation detail. The spec is intentionally stack-neutral and does not specify route precedence rules for any framework. |
| Different detail level in `plan.md §3` vs. `tasks.md T-19` architecture-test rows | Both rows are internally consistent within each spec; the difference between specs reflects different tools (import-linter vs. NetArchTest) and is expected for distinct stacks. |

---

## Note on bifurcations

**Methodology:** `git diff --no-index` was run between the two versions of every shared file.

**Result:**

| File pair | Differences |
|---|---|
| `spec.md` | None — byte-for-byte identical |
| `api-contract.md` | None — byte-for-byte identical |
| `data-model.md` | None — byte-for-byte identical |
| `adr/adr-001` | None — byte-for-byte identical |
| `adr/adr-002` | None — byte-for-byte identical |
| `adr/adr-003` | None — byte-for-byte identical |
| `adr/adr-004` | None — byte-for-byte identical |
| `constitution.md` | Stack-only differences: title, domain path (`src/library/` vs `src/Library.Domain/`), standard library primitives, verification tool (`import-linter` vs `NetArchTest`), migration engine (Alembic vs EF Core), and composition module path |
| `architecture.md` | Stack-only differences: title, HTTP framework label, persistence label, root path casing (`src/library/` vs `src/Library/`), composition file name |
| `plan.md` | Stack-only differences: title, D-01 through D-03 (language, framework, ORM), D-09 (hash algorithm), D-12 (test framework), architecture test tool |

Every difference found is a stack-specific technical choice. No file defines a business rule, API contract, data-model constraint, or ADR decision differently between the two specs.

**Conclusion: 0 bifurcations confirmed by diff.**

---

## Summary

| Type | Count | Findings |
|---|---|---|
| Completeness | 5 | F-02, F-06, F-07, F-11, F-12 |
| Consistency | 1 | F-13 *(possible)* |
| Bifurcation | 0 | — |
| Ambiguity | 6 | F-03, F-04, F-08, F-09, F-10, F-15 |
| **Total** | **12** | |

### By severity

| Severity | Count | Findings |
|---|---|---|
| High | 2 | F-02, F-04 |
| Medium | 8 | F-03, F-06, F-07, F-08, F-09, F-10, F-11, F-12 |
| Low | 2 | F-13, F-15 |

### Top priority before implementation

1. **F-02** — The version-witness column is missing from `data-model.md`. Optimistic concurrency (ADR-002) cannot be implemented without it.
2. **F-04** — The error body when multiple books in a loan are unavailable is underspecified.
3. **F-06** — `GET /books/{id}/cover` is absent from the endpoint table.
4. **F-07** — It is unspecified whether `PUT /books/{id}` can update the ISBN.
5. **F-09** — The date-range filter column for the most-borrowed report is never named.
