# Design pre-review: Make ginInsertCleanup follow vacuum's BufferAccessStrategy

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7177](https://commitfest.postgresql.org/patch/7177/) (Needs review); authors: Kevin Rocker (kehvyn)
- Thread: [\[Patch\] Make ginInsertCleanup follow vacuum's BufferAccessStrategy](https://www.postgresql.org/message-id/flat/76104756-9ca6-45ee-bd16-ead325f1c149%40app.fastmail.com), started 2026-08-18
- Messages read: 3 of 3
- Patch prose read: v2-0001-Use-the-vacuum-buffer-access-strategy-in-GIN-pend.patch, v1-0001-Use-the-vacuum-buffer-access-strategy-in-GIN-pend.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. The patch extends the existing vacuum BufferAccessStrategy to the GIN pending-list reads, adds no user-visible surface, and leaves the insert path and gin_clean_pending_list() unchanged. Confidence is low because the thread is three messages long and shows no measurements.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it is not possible to check that every pending-list read in ginInsertCleanup() and shiftList() receives the strategy, or that the NULL callers behave as before. The effect of the ring buffer on a long pending list is unmeasured.

## The proposal, restated

**Problem.** When VACUUM reaches ginInsertCleanup(), the function reads the metapage and every pending-list page with ReadBuffer() and ignores the vacuum's BufferAccessStrategy. shiftList() does the same when it re-reads processed pages. The author first saw this as an inconsistency, and suggests it also causes extra evictions from shared buffers for other processes.

**Proposed design.** ginInsertCleanup() and shiftList() take a strategy argument and pass it down to ReadBufferExtended. The three vacuum-side callers pass the vacuum strategy. The post-insert cleanup path and gin_clean_pending_list() pass NULL, so their behaviour does not change.

**User-visible surface.**
- C API: ginInsertCleanup() in gin_private.h gains a BufferAccessStrategy argument.

**What the author claims.**
- A test script shows the strategy changes how GIN pending-list cleanup uses the buffer cache.
- It may reduce evictions of other processes' pages from shared buffers.
- Other call sites keep their current behaviour.
- ReadStream, as in 6c228755add, does not fit, because each pending-list page must be read to find the next block.
- It could be backpatched as far as 14, but the author proposes it for master only.

**Thread state.** Xuneng Zhou agreed with the rationale and with the point that ReadStream does not fit, and the author rebased to v2. Nobody has discussed backpatching, and the entry is still in Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | ginInsertCleanup() in gin_private.h gains a BufferAccessStrategy argument. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #3 says a backpatch to 14 is technically possible; a backpatch would change a function signature on stable branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | VACUUM of a GIN index now reads pending-list pages through the vacuum ring buffer instead of plain shared buffers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What does the attached test script show, in buffer hits, evictions and vacuum duration, with and without the strategy?
2. Does the vacuum ring buffer slow cleanup of a long pending list, given that shiftList() re-reads the processed pages before deleting them?
3. Should gin_clean_pending_list() keep passing NULL, and what is the reason for treating it differently from the vacuum callers?
4. Do any known extensions call ginInsertCleanup() and need a change when its signature changes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows GIN fast-update cleanup and the vacuum buffer access strategy code would have to own it. The one reviewer in the thread agreed with the rationale only.
- **Liability:** The owner signs for the buffer-cache behaviour of GIN pending-list cleanup under VACUUM, including any slowdown on long pending lists. The change is small and easy to revert on master.
- **Author follow-up:** The author rebased to v2 five days after v1 and moved the entry to the next CommitFest, which suggests continued attention.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 6.4 KB across three GIN files and adds no user-visible surface, so review is cheap. It benefits installations with large GIN pending lists; numbers from the test script would help a committer decide quickly.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The precedent search returned no earlier proposal with the same or a similar idea. The thread does not point to any earlier discussion beyond commit 6c228755add, which message #1 and message #2 cite as related work, not as a rejected attempt. So no prior rejection reason applies here.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no SQL syntax, GUC, catalog object or psql change. Nothing new needs to round-trip through pg_dump. Existing queries and scripts see no different results, and gin_clean_pending_list() keeps its current behaviour. The only signature change is ginInsertCleanup() in the private header gin_private.h, and the proposal targets master only, so back-branch ABI is not affected. Buffer-cache effects belong to other angles.
- replication-ondisk: 0 objection(s). The patch only changes which buffer access strategy is used when reading GIN pages. It adds no on-disk format, WAL record, catalog, wire protocol or replication change. The ginInsertCleanup() signature change is in the private header gin_private.h and is proposed for master only, so ABI matters only if someone later asks for a backpatch.
- security: 0 objection(s). The patch has no privilege or exposure aspect. It passes the existing vacuum BufferAccessStrategy into GIN pending-list cleanup, changes no user-callable interface or permission check, and leaves gin_clean_pending_list() and the insert path unchanged. A ring buffer limits shared-buffer use, so it adds no denial-of-service path for unprivileged users.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection. The patch follows the existing practice of passing the vacuum strategy to index reads, adds no per-tuple or per-query cost for anyone outside VACUUM, and changes only a function declared in gin_private.h, on master only. The open points are measurement questions for the performance angle: no numbers are given, and the effect of a small ring on a long pending list is unknown.
- already-exists: 0 objection(s). No existing feature covers this. The pending-list reads ignore the vacuum strategy, so vacuum_buffer_usage_limit has no effect on them. The patch extends the existing BufferAccessStrategy mechanism and adds no second one, and an extension could not change how core GIN reads these pages.
- demand: 0 objection(s). The material shows no field reports or numbers. But this is a consistency fix that makes GIN pending-list cleanup follow the same vacuum BufferAccessStrategy other index vacuum code uses. It adds no user-visible surface, and its only cost is a C signature change, so a demand objection does not apply.

### Precedents retrieved

- P1: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant; reason summarised from its thread
- P2: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P3: [Revert "Add eager and lazy freezing strategies to VACUUM."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c6b4972664491de408670354fdbccc7c83a1c89) (revert, Reverted, 2023-01-26) - judged not relevant
- P4: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P5: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P6: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P7: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P8: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant; reason summarised from its thread
- P9: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P10: [pg_restore cleanups](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=40b9c27014d90d77c61e8f5c77ddeb84fa9a6b69) (revert, Partially reverted, 2025-04-16) - judged not relevant
- P11: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P12: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant
- P13: [Revert exports.list change pending closer study.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=36b870623698332144c73e950d6a8f91e883d708) (revert, Reverted, 2006-12-28) - judged not relevant
- P14: [Revert error message on GLOBAL/LOCAL pending further discussion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28ac7972873bd314d4837807396fe44571b5cb50) (revert, Reverted, 2012-06-10) - judged not relevant
- P15: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P16: [Revert patch pending more discussion:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=782df58a1cafbef5defb329a7583dfa0154dfe72) (revert, Reverted, 2006-04-30) - judged not relevant
- P17: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P18: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P19: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P20: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P21: [\[BUG\] Assert fails for local buffers in BufferIsExclusiveLocked](https://commitfest.postgresql.org/patch/5427/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P22: [Revert no-op changes to BufferGetPage()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a343e223a5c33a7283a6d8b255c9dbc48dbc5061) (revert, Reverted, 2016-04-20) - judged not relevant
- P23: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P24: [Reset vacuum_defer_cleanup_age to PGC_SIGHUP. Revert commit 84725aa5efe11688633b553e58113efce4181f2e](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f480e294498533820f3ef3e6de4dcb8ff5401140) (revert, Partially reverted, 2013-02-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 57 s; cost reported by the CLI: $0.59
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:25:25.584Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
