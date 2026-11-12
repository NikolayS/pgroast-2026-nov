# Design pre-review: Use C11 alignas instead of palloc/malloc for alignment

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7338](https://commitfest.postgresql.org/patch/7338/) (Needs review); authors: Peter Eisentraut (petere)
- Thread: [Use C11 alignas instead of palloc/malloc for alignment](https://www.postgresql.org/message-id/flat/94a128da-bf3e-46bd-9e2d-609573c484da%40eisentraut.org), started 2026-09-08
- Messages read: 11 of 11
- Patch prose read: v4-0001-Rename-PGAlignedXLogBlock-to-PGIOAlignedXLogBlock.patch, v4-0002-Use-C11-alignas-instead-of-palloc-malloc-for-alig.patch, v4-0003-XXX-Reintroduce-PGAlignedXLogBlock-and-use-it.patch, v4-0004-Add-an-assertion-and-comments-for-HEADER_SCRATCH_.patch, v4-0005-Avoid-static-variables-in-XLogRecordAssemble.patch, 0001-Use-C11-alignas-instead-of-palloc-malloc-for-alignme.patch, 0002-pgindent-Fix-indentation-of-alignas-in-struct-member.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal, and none is fatal. Three fixable points remain. standards-compat-2 asks whether the rename in 0001 is worth a compile break for the out-of-tree users in message #8. standards-compat-1 shows that 0003 reuses the old name with 8-byte alignment. maintenance-fit-1 notes that the per-record memset in 0005 has no timing.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the memset in 0005 covers all padding, that the new assertion matches the HEADER_SCRATCH_SIZE calculation, or how much stack each converted call site uses. Only 3 of 11 messages were read directly.

## The proposal, restated

**Problem.** Several places call palloc() or malloc() only to get an aligned buffer, which the author finds more complex than needed. The thread names no user-facing problem; this is internal refactoring. Later versions also target static variables in XLogRecordAssemble(), which get in the way of multi-threading.

**Proposed design.** The patch replaces those allocations with local variables declared with alignas(MAXIMUM_ALIGNOF) or with existing AlignedBlock types. It renames PGAlignedXLogBlock to PGIOAlignedXLogBlock, and an optional XXX patch reintroduces PGAlignedXLogBlock as a MAXALIGNed type. It also adds an assertion for HEADER_SCRATCH_SIZE and moves the XLogRecordAssemble() workspace into a zeroed local in XLogInsert().

**User-visible surface.**
- C API: type PGAlignedXLogBlock renamed to PGIOAlignedXLogBlock in c.h; extensions using the old name fail to compile.
- C API: optional patch 0003 reintroduces PGAlignedXLogBlock with MAXALIGN instead of PG_IO_ALIGN_SIZE alignment.
- C API: XLogReader struct definition in xlogreader.h changes.

**What the author claims.**
- Using alignas with local variables is much simpler than palloc/malloc for aligned buffers.
- Previous alignment guarantees are kept; MAXIMUM_ALIGNOF and max_align_t should be the same in practice.
- The copy_file() and InitWalRecovery() allocations stay dynamic, for buffer size and storage reasons.
- Reusing the name PGAlignedXLogBlock for a different alignment without a gap between releases is risky, so 0003 is marked XXX.

**Thread state.** Heikki, Andres and Andrew commented; the author addressed them in v3/v4, and the pgindent fix was committed separately. Manuel tested both 0003 options and the WAL padding, Rui found no issues, and Linden suggested static const zero buffers; whether to apply 0003 and Linden's suggestion are still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Renaming PGAlignedXLogBlock breaks known extensions for naming consistency only

fixable · level 1b, rubric item 12 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Maintainers of pg_tde, pg_flashback, WAL-DIFF and similar tools build against the next major release. Compilation fails on PGAlignedXLogBlock, and they must add version-conditional code although nothing in their behaviour changes.

Message #8 lists at least seven out-of-tree projects that use the type for plain WAL page buffers. Patch 0001 gives them no functional gain; its stated reason is symmetry with PGAlignedBlock and PGIOAlignedBlock. A compatibility alias would keep them building, but it would also block the later MAXALIGNed reuse of the name. The proposal does not weigh these options. A committer would likely ask whether the naming symmetry is worth a break that the project chooses on purpose.

Thread: [#7 (Peter Eisentraut, 2026-09-16)](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8 (Manuel Reyes Bravo, 2026-09-16)](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com)

**Question:** Why is a rename without a compatibility alias better for the projects in message #8 than keeping the old name and adding a separate name for a MAXALIGNed block?

Counter-check: held. Compile breaks with a named replacement are normal across major versions, so the cost is small. But the series does not need 0001, and the thread does not weigh keeping the old name and choosing a new name for a MAXALIGNed block.

### 2. Patch 0003 reuses an existing type name with weaker alignment

fixable · level 2, rubric item 6 · ABI / extension API, backpatchability · `standards-compat-1`

An extension declares PGAlignedXLogBlock and reads WAL with O_DIRECT. After upgrade it still compiles, but the buffer is only 8-byte aligned. pread fails with EINVAL on ext4 and works on btrfs, so the failure shows up only on some customer systems.

Message #8 measured this: with 0001-0005, alignof(PGAlignedXLogBlock) drops from 4096 to 8, and an O_DIRECT read fails on ext4 but succeeds on btrfs. Message #7 already doubts reusing the name without a gap, and 0003 is still marked XXX. The same type name gives a different contract with no compile-time signal. Manuel found no direct-I/O user, but a GitHub search on default branches is only a lower bound.

Thread: [#7 (Peter Eisentraut, 2026-09-16)](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8 (Manuel Reyes Bravo, 2026-09-16)](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com)

**Question:** If 0003 goes in, how many releases will separate the rename from the reintroduced name, and what tells a direct-I/O user that the alignment changed?

Counter-check: held. Message #8 measured the case: with 0003 the old name compiles, and an O_DIRECT read fails with EINVAL on ext4. Message #7 shares the doubt, but v4 still carries 0003 and states no gap, so the question is open.

### 3. XLogInsert() now zeroes a 952-byte workspace on every WAL record

fixable · level 2, rubric item 8 · performance on the hot path, WAL / replication / standby · `maintenance-fit-1`

Every backend that writes WAL now zeroes a 952-byte stack workspace in XLogInsert() for each record, where before a static buffer was zeroed once. A small-record, high-insert workload pays this cost on every call, and nobody has measured it. In _mdfd_getseg(), each call also fills an 8 kB const zero buffer, and the stack frame grows to about 20 kB.

Patch 0005 adds the memset because the static variable used to be zero-initialized for free. Message #8 shows the memset is needed, since without it 16033 WAL records carry uninitialized padding bytes. No message measures the cost on the XLogInsert() path, which every WAL-writing user takes. Message #11 shows the _mdfd_getseg() and dsm_impl_mmap() const locals copy or clear 8 kB per call and suggests static const; v4 does not take that up.

Thread: [#7 (Peter Eisentraut, 2026-09-16)](https://www.postgresql.org/message-id/9e81a344-b927-41e2-acea-da444a03aaa0%40eisentraut.org), [#8 (Manuel Reyes Bravo, 2026-09-16)](https://www.postgresql.org/message-id/CA%2BbCEdCnFaAhpfzYRh4cWCRUYCxMjxCM%2BV3_KXau_1pYhbDTbQ%40mail.gmail.com), [#9 (Peter Eisentraut, 2026-10-19)](https://www.postgresql.org/message-id/8bdc9593-cbe1-4c61-abe0-732cc8724acb%40eisentraut.org), [#11 (best xmg, 2026-11-08)](https://www.postgresql.org/message-id/CABkqMMu-DpTsAnmwwmuFNuqeYmrAPfYZS8WjJtEhbPSdNVJ5Bg%40mail.gmail.com)

**Question:** What does a pgbench or small-insert benchmark show for v4 0005 against master, and will the 8 kB zero buffers become static const?

Counter-check: held. Patch 0005 adds a memset of the workspace on every XLogInsert() call, and the thread shows no timing for it. Message #11's static const suggestion for the 8 kB zero buffers has no reply yet.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Patches 0004 and 0005 change how xloginsert.c builds the WAL record header; 0002 changes buffers in xlogreader.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 renames PGAlignedXLogBlock in c.h, 0003 reuses the name with weaker alignment, and the XLogReader struct in xlogreader.h changes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | With 0003, PGAlignedXLogBlock means a different alignment in new and old branches, so back-patched code using the name behaves differently. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The pgindent workaround for alignas and the HEADER_SCRATCH_SIZE assertion are small new pieces to keep correct. |
| performance on the hot path | **yes** | 0005 zeroes a 952-byte local workspace in XLogInsert() for every WAL record. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will 0003 be part of the commit, and if so, how many major releases will separate the rename in 0001 from the reuse of the name PGAlignedXLogBlock?
2. Why is renaming PGAlignedXLogBlock without a compatibility alias better than keeping the old name and choosing a new name for a MAXALIGNed block?
3. What does a small-record insert benchmark show for v4 with 0005 against master?
4. Will the zero buffers in _mdfd_getseg() and dsm_impl_mmap() become static const, as message #11 suggests?
5. Which call sites does 0002 convert, and how much stack does each one add?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The author is listed as Peter Eisentraut (petere), and Heikki Linnakangas reviewed the WAL insert part in message #2. The owner needs to know the WAL insert path and the direct I/O alignment rules.
- **Liability:** The owner signs for zeroed padding in every WAL record header, for the stack use of the converted call sites, and for the meaning of PGAlignedXLogBlock in extension code across major releases.
- **Author follow-up:** The author answered the review comments in v3 and v4 and added the two patches that Heikki suggested. Message #11 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** This is internal refactoring that helps core developers and the multi-threading work, and it adds no user-visible surface apart from the type rename. The patches are small, but 0005 changes the WAL insert path, which needs a careful reader and a timing.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Reusing PGAlignedXLogBlock with weaker alignment breaks direct I/O silently. Refuted (not specific): Duplicate of standards-compat-1. Both describe the same O_DIRECT failure from 0003 and cite the same measurements in message #8.
- `maintenance-fit-2` Renaming PGAlignedXLogBlock breaks known extensions for naming consistency only. Refuted (not specific): Duplicate of standards-compat-2 for the rename and of standards-compat-1 for the later reuse of the name. It adds no separate scenario.
- `demand-1` Rename of PGAlignedXLogBlock costs extension authors for naming consistency only. Refuted (not specific): Duplicate of standards-compat-2. The cost to the projects in message #8 and the option of keeping the old name are covered there.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only retrieved row, [P17], reverted a C11 change because standard C++ has no restrict; C++11 has alignas as a keyword, so that reason does not carry over to the alignas use in xlogreader.h. The thread points to no earlier discussion of this idea.
- standards-compat: 2 objection(s). The other changes are internal refactoring. They add no SQL syntax, GUC, catalog or dump behaviour. Message #8 shows WAL contents are unchanged.
- replication-ondisk: 1 objection(s). WAL content is unchanged: message #8 shows the memset in 0005 keeps the XLogRecord padding zeroed. The patch has no on-disk, catalog, or protocol change. The xlogreader.h struct change and the 0001 rename are normal API breaks in a major release.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It is internal refactoring of buffer allocation, and no user gains new rights, data or resources. The one possible exposure is uninitialized stack bytes in XLogRecord padding reaching the WAL, where replication and pg_waldump users can read them. Patch 0005 adds a memset for this, and message #8 measured zero non-zero padding bytes with it.
- maintenance-fit: 2 objection(s). Two objections stand. Experienced committers wrote and reviewed this internal refactoring, so the maintenance-ownership risk is low.
- already-exists: 0 objection(s). Nothing in core or in an extension already does this job: the patch is internal refactoring that reuses existing mechanisms (C11 alignas, PGAlignedBlock, PGIOAlignedBlock), so no second way is added. The one existing idiom it could follow better is the static const zero buffer in pg_pwrite_zeros(), raised in message #11; that is an open review point, not a reason to reject.
- demand: 1 objection(s). The core refactoring (alignas locals, the HEADER_SCRATCH_SIZE assertion, removing statics in XLogRecordAssemble) is internal cleanup with no user-facing cost. Three reviewers support it and it helps the multi-threading work, so demand is not a valid objection to it.

### Precedents retrieved

- P1: [Cache line alignment of PGXACT](https://commitfest.postgresql.org/patch/974/) (cf, Returned with feedback, 2017-09-22) - judged not relevant; reason summarised from its thread
- P2: [change SOFT_ERROR_OCCURRED to a static inline function](https://commitfest.postgresql.org/patch/6666/) (cf, Withdrawn, 2026-08-07) - judged not relevant; reason summarised from its thread
- P3: [Get rid of the pre-C11 _Alignof define in zic.c](https://commitfest.postgresql.org/patch/6453/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P4: [Mingw: Fix import library extension, build actual static libraries](https://commitfest.postgresql.org/patch/2082/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P6: [Revert "Speed up tail processing when hashing aligned C strings"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4ad0021aface01ee74604355bd2e998e5f73320) (revert, Reverted, 2024-03-31) - judged not relevant
- P7: [Align verify_heapam.c error message offset with test expectations](https://commitfest.postgresql.org/patch/6423/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P8: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant; reason summarised from its thread
- P9: [Fractions in GUC variables](https://commitfest.postgresql.org/patch/86/) (cf, Rejected, 2015-01-12) - judged not relevant; reason summarised from its thread
- P10: [psql - add :SERVER_VERSION variable](https://commitfest.postgresql.org/patch/1291/) (cf, Rejected, 2017-11-13) - judged not relevant
- P11: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant
- P12: [add assertion for palloc in signal handlers](https://commitfest.postgresql.org/patch/6498/) (cf, Withdrawn, 2026-08-24) - judged not relevant
- P13: [rename column if exists](https://commitfest.postgresql.org/patch/3045/) (cf, Rejected, 2022-01-17) - judged not relevant
- P14: [block level PRAGMA](https://commitfest.postgresql.org/patch/1925/) (cf, Rejected, 2019-04-09) - judged not relevant
- P15: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P16: [Avoid pallocs in async.c's "critical section"](https://commitfest.postgresql.org/patch/6248/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P17: [Revert "Replace pg_restrict by standard restrict"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fa16e7fd84886643f480c36614fa11d45fc6d26f) (revert, Reverted, 2026-01-14) - used in the review
- P18: [Revert "Revert part of commit dddf4cdc3."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ee201520707596b058e6bbfab0a5e2006f3834c5) (revert, Reverted, 2019-10-25) - judged not relevant
- P19: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P20: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant
- P21: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P22: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/1223/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P23: [pg_stat_lock: add blocker mode dimension](https://commitfest.postgresql.org/patch/6820/) (cf, Withdrawn, 2026-08-13) - judged not relevant
- P24: [Rename "getdatabaseencoding()" to "pg_database_encoding()", and document](https://commitfest.postgresql.org/patch/7024/) (cf, Withdrawn, 2026-07-18) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 145 s; cost reported by the CLI: $1.73
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 75 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:41:52.196Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
