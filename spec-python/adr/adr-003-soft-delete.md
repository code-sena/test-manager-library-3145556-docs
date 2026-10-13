# ADR-003 · Books get a soft delete, not a hard delete

- **Status:** accepted
- **Context:** a loan line points to a book. If the book is physically deleted, the history and the
  report break.
- **Decision:** the book carries `deleted_at`. Deleting it sets that column; the catalog filters by
  `deleted_at IS NULL` in a single place (a global filter). A book with open loans is **not** deleted
  (409 `BOOK_HAS_OPEN_LOANS`). ISBN uniqueness is a **partial** index over active books, so a deleted
  ISBN can be registered again.
- **Alternatives discarded:** physical cascading delete (destroys the history); an `active = false`
  flag (does not say when).
- **Consequences:** `loan_item.book_id` carries a `RESTRICT` FK as a **last-resort barrier**: a manual
  `DELETE` or a future code change fails loudly instead of corrupting the history. The cover binary is
  the only data that is actually deleted physically.
- **How it is verified:** looking up a deleted book returns 404; a `DELETE` through `psql` on a book
  with lines fails because of the FK.
