# Design pre-review: Faulty pg_get_shmem_pagesize() on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7563](https://commitfest.postgresql.org/patch/7563/); [thread](https://www.postgresql.org/message-id/flat/CAExHW5vuMVLMOf28aD75YKS66gyu3d55%2Bafk4G7igGFWwuQvHQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No objection survived at levels 1a or 1b: the problem is stated, and 0001 fixes a latent bug at small cost. The one surviving objection, maintenance-fit-1, is a fixable design point at level 2 about 0002 having no loud failure when an allocation path does not record the page size. Only message #1 is visible and no code was read, so confidence is low.

Not checked: Without the code, nobody checked where the new field sits in PGShmemHeader, whether each allocation path sets it, or whether the Windows path behaves correctly with large pages.

## Prior attempts

None found before 2026-10-30.

## Objections a reviewer is likely to raise

### 1. Recorded page size has no check when an allocation path skips it

A later patch adds or changes an allocation path in sysv_shmem.c, for example for resizable shared memory, and does not set the new PGShmemHeader field. pg_get_shmem_pagesize() then returns zero or a stale value, and no test or build fails. Callers that compute alignment or page counts from it go wrong, and nobody notices until a user reports odd NUMA output or a resize failure.

Message #1 says 0002 relies on "every method of allocating shared memory will continue to record page size", so correctness moves from one accessor to each allocation path. sysv_shmem.c has several of these paths (huge-page mmap, ordinary mmap with the huge_pages=try fallback, System V), and win32_shmem.c has its own, so a future change has several places to miss. The proposal does not describe an assertion or other loud failure when the field is unset. shared_memory_size_in_huge_pages still uses GetHugePageSize(), so some duplicated page-size logic stays.

**Question:** What makes pg_get_shmem_pagesize() fail loudly, for example an assertion on a zero value, if a current or future allocation path does not record the page size?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAExHW5vuMVLMOf28aD75YKS66gyu3d55%2Bafk4G7igGFWwuQvHQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What makes pg_get_shmem_pagesize() fail loudly, for example an assertion on a zero value, if a current or future allocation path does not record the page size?
2. How does sysv_shmem.c record the page size for each allocation method: huge-page mmap, ordinary mmap after the huge_pages=try fallback, and System V?
3. How was 0002 tested on Windows with large pages in use?
4. Does the new PGShmemHeader field leave the offsets of the existing fields unchanged, and does any extension or tool read the header layout directly?
5. If 0002 is committed, is 0001 still needed as a separate commit?

## Touch points

- **ABI / extension API:** 0002 adds a field to PGShmemHeader in pg_shmem.h and changes what the exported pg_get_shmem_pagesize() returns on Windows.
- **backpatchability:** Message #1 proposes master only, with no backpatch, because no current caller reaches the faulty path on Windows.
- **maintenance burden ("who maintains this in 10 years"):** After 0002, every allocation path in sysv_shmem.c and win32_shmem.c must record the page size in PGShmemHeader.

## Who would commit it, and when

- Owner: silent. Someone who knows the shared memory creation code in sysv_shmem.c and win32_shmem.c, including huge pages and Windows large pages. No committer has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. No user sees the bug today; the benefit goes to future callers on Windows and to resizable shared memory. The patches are small (1.8 KB and 6.4 KB), touch four files in the shared memory code, and add one header field.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
