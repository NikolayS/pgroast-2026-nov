# Design pre-review: Faulty pg_get_shmem_pagesize() on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7563](https://commitfest.postgresql.org/patch/7563/) (Needs review); authors: Ashutosh Bapat (ashutoshbapat)
- Thread: [Faulty pg_get_shmem_pagesize() on Windows](https://www.postgresql.org/message-id/flat/CAExHW5vuMVLMOf28aD75YKS66gyu3d55%2Bafk4G7igGFWwuQvHQ%40mail.gmail.com), started 2026-10-30
- Messages read: 1 of 1
- Patch prose read: v20261030-0001-pg_get_shmem_pagesize-on-Windows.patch, v20261030-0002-Record-the-main-shared-memory-segment-s-pa.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No objection survived at levels 1a or 1b: the problem is stated, and 0001 fixes a latent bug at small cost. The one surviving objection, maintenance-fit-1, is a fixable design point at level 2 about 0002 having no loud failure when an allocation path does not record the page size. Only message #1 is visible and no code was read, so confidence is low.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked where the new field sits in PGShmemHeader, whether each allocation path sets it, or whether the Windows path behaves correctly with large pages.

## The proposal, restated

**Problem.** pg_get_shmem_pagesize() returns zero on Windows when large pages are in use, because it calls GetHugePageSize(), which is a stub there. The only current caller, NUMA shared memory reporting, rejects Windows first, so no user sees the bug today. The author wants the function to be correct for future callers, including resizable shared memory.

**Proposed design.** 0001 calls GetLargePageMinimum() on Windows when large pages are in use, and leaves the GetHugePageSize() stub and shared_memory_size_in_huge_pages unchanged. 0002 stores the page size in PGShmemHeader when the main segment is created, including the ordinary-page fallback with huge_pages=try. pg_get_shmem_pagesize() then returns the stored value, which 0002 sets again when shared memory is recreated.

**User-visible surface.**
- C API: pg_get_shmem_pagesize() returns the large-page size on Windows instead of zero.
- C API: PGShmemHeader in pg_shmem.h gains a field for the page size of the main segment.

**What the author claims.**
- No current user-visible failure, so the fix targets master only with no backpatch.
- 0002 removes duplicated platform logic, so the accessor cannot disagree with the allocation code.
- 0002 avoids opening and scanning /proc/meminfo on every call on Linux, though the author measured no significant gain for the NUMA caller.
- Resizable shared memory will need the page size during resizing and protection changes.
- Earlier threads did not discuss the Windows stub returning zero or storing the page size in the header.

**Thread state.** Only the first message is visible. Nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Recorded page size has no check when an allocation path skips it

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds or changes an allocation path in sysv_shmem.c, for example for resizable shared memory, and does not set the new PGShmemHeader field. pg_get_shmem_pagesize() then returns zero or a stale value, and no test or build fails. Callers that compute alignment or page counts from it go wrong, and nobody notices until a user reports odd NUMA output or a resize failure.

Message #1 says 0002 relies on "every method of allocating shared memory will continue to record page size", so correctness moves from one accessor to each allocation path. sysv_shmem.c has several of these paths (huge-page mmap, ordinary mmap with the huge_pages=try fallback, System V), and win32_shmem.c has its own, so a future change has several places to miss. The proposal does not describe an assertion or other loud failure when the field is unset. shared_memory_size_in_huge_pages still uses GetHugePageSize(), so some duplicated page-size logic stays.

Thread: [#1 (Ashutosh Bapat, 2026-10-30)](https://www.postgresql.org/message-id/CAExHW5vuMVLMOf28aD75YKS66gyu3d55%2Bafk4G7igGFWwuQvHQ%40mail.gmail.com)

**Question:** What makes pg_get_shmem_pagesize() fail loudly, for example an assertion on a zero value, if a current or future allocation path does not record the page size?

Counter-check: held. Message #1 itself says 0002 depends on "hoping that every method of allocating shared memory will continue to record page size". The patch prose names no assertion or other loud failure for an unset field, so the request for one stands as a fixable point.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0002 adds a field to PGShmemHeader in pg_shmem.h and changes what the exported pg_get_shmem_pagesize() returns on Windows. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 proposes master only, with no backpatch, because no current caller reaches the faulty path on Windows. |
| maintenance burden ("who maintains this in 10 years") | **yes** | After 0002, every allocation path in sysv_shmem.c and win32_shmem.c must record the page size in PGShmemHeader. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What makes pg_get_shmem_pagesize() fail loudly, for example an assertion on a zero value, if a current or future allocation path does not record the page size?
2. How does sysv_shmem.c record the page size for each allocation method: huge-page mmap, ordinary mmap after the huge_pages=try fallback, and System V?
3. How was 0002 tested on Windows with large pages in use?
4. Does the new PGShmemHeader field leave the offsets of the existing fields unchanged, and does any extension or tool read the header layout directly?
5. If 0002 is committed, is 0001 still needed as a separate commit?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the shared memory creation code in sysv_shmem.c and win32_shmem.c, including huge pages and Windows large pages. No committer has replied in the visible thread.
- **Liability:** The owner signs for the rule that every allocation path records the page size in PGShmemHeader, for as long as the accessor reads it. Resizable shared memory would add callers that depend on it.
- **Author follow-up:** The author says he found this while working on resizable shared memory, which will use the function, so he has a reason to stay. The thread has no replies yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** No user sees the bug today; the benefit goes to future callers on Windows and to resizable shared memory. The patches are small (1.8 KB and 6.4 KB), touch four files in the shared memory code, and add one header field.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The earlier discussions cited in message #1 left shared_memory_size_in_huge_pages at -1 on Windows, and this proposal keeps that GUC and the GetHugePageSize() stub as they are, so the reason recorded there does not apply. Message #1 reports that no earlier thread discussed the stub returning zero or storing the page size in PGShmemHeader.
- standards-compat: 0 objection(s). Neither patch touches SQL syntax, GUCs, pg_dump, psql, the wire protocol or on-disk data. The only contract change is a new field in PGShmemHeader, an in-memory structure that is rebuilt at each start, and the patches target master only, so the ABI rule for minor releases does not apply. shared_memory_size_in_huge_pages keeps its existing Windows behaviour, as earlier threads agreed. Without the code I could not check where the new field sits in PGShmemHeader or whether any extension reads the header layout directly.
- replication-ondisk: 0 objection(s). The patches change only in-memory state: PGShmemHeader lives in the main shared memory segment and is rebuilt at each start and after crash reinitialization. Nothing reaches WAL, relation files, catalogs, pg_upgrade or the wire protocol. The header change is master-only, and extensions rarely read PGShmemHeader directly. Without the code, I could not check that the new field leaves the offsets of magic, creatorPID and the device/inode fields unchanged, which stale-segment detection reads.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes an internal C accessor and stores a page size in PGShmemHeader. The page size is already readable by any user through pg_shmem_allocations_numa and huge_page_size, and the patch adds no SQL-callable surface, role, or new data a non-superuser could reach.
- maintenance-fit: 1 objection(s). Beyond the item above, the maintenance cost is small: a master-only bugfix with one header field, no hot-path cost, and no new external dependency. Need for 0002 rests on unmerged resizable shared memory, but the change is cheap enough that this alone is not an objection.
- already-exists: 0 objection(s). Postgres has no existing feature, extension or tool that reports the page size of the main shared memory segment, including on Windows; huge_pages_status reports only whether huge pages are in use. Placing the size in PGShmemHeader extends the existing accessor and adds no second mechanism, so this angle has no objection.
- demand: 0 objection(s). There is no demand objection to raise. 0001 fixes a latent bug: an exported accessor that returns zero on Windows with large pages, at small cost and with no new user surface. 0002 rests partly on resizable shared memory, which is not yet committed, but its cost is one header field that only the accessor reads, and it removes duplicated platform logic.

### Precedents retrieved

- P1: [Supporting huge pages on Windows](https://commitfest.postgresql.org/patch/813/) (cf, Returned with feedback, 2017-04-07) - judged not relevant; reason summarised from its thread
- P2: [Auto-tune shared_buffers to use available huge pages](https://commitfest.postgresql.org/patch/6427/) (cf, Withdrawn, 2026-06-24) - judged not relevant; reason summarised from its thread
- P3: [Identify huge pages accessibility using madvise](https://commitfest.postgresql.org/patch/4937/) (cf, Withdrawn, 2024-11-12) - judged not relevant; reason summarised from its thread
- P4: [Support huge_pages on AIX](https://commitfest.postgresql.org/patch/1960/) (cf, Withdrawn, 2019-12-03) - judged not relevant; reason summarised from its thread
- P5: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P7: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant; reason summarised from its thread
- P8: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P10: [Added support of XML_PARSE_HUGE flag for XML documents](https://commitfest.postgresql.org/patch/4772/) (cf, Rejected, 2024-01-31) - judged not relevant
- P11: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P12: [Fix severe performance regression with gettext 0.20+ on Windows](https://commitfest.postgresql.org/patch/6491/) (cf, Returned with feedback, 2026-03-27) - judged not relevant
- P13: [Revert recent change to RequestNamedLWLockTranche().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d814d7fc3d5257ae258b502229fc7ca97c97270a) (revert, Reverted, 2025-09-04) - judged not relevant
- P14: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P15: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P16: [Make pg_numa_init() cope with Docker](https://commitfest.postgresql.org/patch/6171/) (cf, Withdrawn, 2026-01-29) - judged not relevant
- P17: [Fix LWLock degradation on NUMA](https://commitfest.postgresql.org/patch/1166/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P18: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P19: [Large files for relations](https://commitfest.postgresql.org/patch/4305/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [a large LIMIT makes some sorts slower](https://commitfest.postgresql.org/patch/7084/) (cf, Withdrawn, 2026-10-26) - judged not relevant
- P21: [Get rid of an extra error for client disconnection on Windows](https://commitfest.postgresql.org/patch/659/) (cf, Rejected, 2016-09-13) - judged not relevant
- P22: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P23: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P24: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 76 s; cost reported by the CLI: $0.97
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 1 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:10:34.550Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
