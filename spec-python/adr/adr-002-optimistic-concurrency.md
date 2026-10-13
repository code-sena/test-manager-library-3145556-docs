# ADR-002 · Optimistic concurrency over the copies

- **Status:** accepted
- **Context:** two librarians can lend the **last copy** of the same book at the same time (NFR-2).
  If both read `available_copies = 1` and both write, the system lends a book it does not have.
- **Decision:** **optimistic** concurrency control. The book carries a version witness; on save, if
  the version changed, the operation fails and the server responds **409 `BOOK_NOT_AVAILABLE`** (or
  retries once if a copy is still available). The **last line of defense** is the `CHECK`
  `ck_book_available_range` (`0 <= available_copies <= total_copies`).
- **Alternatives discarded:** pessimistic row locking (`SELECT … FOR UPDATE`) on every loan:
  serializes the whole library for a rare case; an in-memory counter: does not survive more than one
  process.
- **Consequences:** if the `CHECK` trips, **something wrote outside the adapter**: it is a defect,
  not a business case. Returning the same line twice does not add twice because `returned_at` is
  already filled in (AC-07.2).
- **How it is verified:** a concurrent test with two simultaneous requests on a book with one copy:
  exactly one gets 201.
