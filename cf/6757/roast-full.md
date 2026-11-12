# Design pre-review: Avoid unnecessary StringInfo allocation in tablesync COPY buffer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6757](https://commitfest.postgresql.org/patch/6757/) (Needs review); authors: Chao Li (chaol)
- Thread: [Avoid unnecessary StringInfo allocation in tablesync COPY buffer](https://www.postgresql.org/message-id/flat/5B2C9B4C-EAE6-4F21-AF99-613A561D26DC%40gmail.com), started 2026-05-09
- Messages read: 5 of 5
- Patch prose read: v2-0001-Use-simple-struct-for-table-sync-COPY-buffer-stat.patch, v1-0001-Avoid-unnecessary-StringInfo-allocation-in-tables.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The change is a refactoring of one file, tablesync.c, with no user-visible surface. P20 does not carry over, because v3 uses initReadOnlyStringInfo(), which is meant for buffers that are not NUL-terminated. The open point is a choice of form between v2 (CopyBuf struct) and v3 (read-only StringInfoData), which Álvaro Herrera has not yet answered.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and v3 has no commit message or file list in the material. Whether v3 resets the static copybuf before each COPY and keeps leftover data valid between calls could not be checked.

## The proposal, restated

**Problem.** In tablesync.c, copy_table() allocates copybuf with makeStringInfo(), but copy_read_data() uses it only to hold a data pointer, length and cursor. The allocated buffer is never used, because walrcv_receive() supplies the data. The thread names no user-facing problem; this is a code cleanup.

**Proposed design.** v1 replaces the heap allocation with a static StringInfoData that is reset before COPY starts. v2 used a small file-local CopyBuf struct instead, after Álvaro Herrera called the StringInfo use a misuse. v3, by Zizhuan Liu, makes copybuf a function-scope static StringInfoData in copy_read_data() and wraps each received buffer with initReadOnlyStringInfo().

**What the author claims.**
- The buffer from makeStringInfo() is never used, so both allocations can go.
- The change follows earlier cleanups that prefer stack-allocated StringInfoData, such as commit a63bbc811d41.
- Denis Smirnov expects no measurable performance difference between v1 and v2.
- v3 keeps the existing handling of leftover unconsumed data across calls.
- Read-only StringInfoData via initReadOnlyStringInfo() is an established pattern in walreceiver.c, logical/worker.c and other files.

**Thread state.** Álvaro Herrera objected to v1, Chao Li answered with v2, and Denis Smirnov argued for v1 with initReadOnlyStringInfo(). Zizhuan Liu, a listed reviewer, posted v3 on that line; neither Álvaro nor Chao Li has replied to v3.

## 2. Prior attempts

### Revert "Optimize various aggregate deserialization functions" (similar idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAApHDvorfO3iBZ=xpiZvp3uHtJVLyFaPBSvcAhAq2HPLnaNSwQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAApHDvorfO3iBZ=xpiZvp3uHtJVLyFaPBSvcAhAq2HPLnaNSwQ@mail.gmail.com), [https://www.postgresql.org/message-id/flat/CAApHDvr=e-YOigriSHHm324a40HPqcUhSp6pWWgjz5WwegR=cQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAApHDvr=e-YOigriSHHm324a40HPqcUhSp6pWWgjz5WwegR=cQ@mail.gmail.com)
- **Date:** decided 2023-10-10; started 2023-10-09
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 608fd198def5390c3490bfe903730207dfd8eeb4. On 2nd thoughts, the StringInfo API requires that strings are NUL terminated and pointing directly to the data in a bytea Datum isn't NUL terminated.
- **Reason (quote):** "On 2nd thoughts, the StringInfo API requires that strings are NUL terminated and pointing directly to the data in a bytea Datum isn't NUL terminated."
- **What is different this time:** P20 pointed a regular StringInfo at bytea data that is not NUL-terminated. v3 wraps the received buffer with initReadOnlyStringInfo(), the read-only form meant for such externally owned buffers.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which version is now proposed for commit: v2 with the file-local CopyBuf struct, or v3 with a read-only StringInfoData?
2. Does wrapping each received buffer with initReadOnlyStringInfo() answer the "misuse of the StringInfo abstraction" concern from message #2?
3. Is the static copybuf in v3 reset before each COPY, including when one worker syncs a second table?
4. Will v3 get a commit message that explains why a static variable is used and how leftover data survives between calls?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera objected to v1 in message #2 but has not replied to v3. A committer who knows the logical replication tablesync code would have to pick between v2 and v3.
- **Liability:** Small: one function's buffer state in tablesync.c. The owner signs that leftover COPY data and the static state behave as before in every tablesync worker.
- **Author follow-up:** Chao Li posted v2 two days after the objection in message #2. The thread shows no reply from him to v3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is under 3 KiB, touches only tablesync.c and benefits the people who maintain that file. It has waited through three CommitFests for agreement on v2 or v3, which one reply from the objecting committer could settle.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only related precedent, P20, was reverted because a StringInfo pointed at a buffer that was not NUL-terminated. v3 uses initReadOnlyStringInfo(), which exists for that case, so the reason does not carry over. Álvaro's "misuse" objection (message #2) is still open on v3, but it is a design-taste question that no precedent decides.
- standards-compat: 0 objection(s). The change is internal to tablesync.c and has no user-facing surface: no syntax, GUC, catalog, pg_dump, psql or client-visible behaviour. Nothing in the standards and compatibility angle applies. P20 is about NUL termination in read-only StringInfo, which is an implementation concern outside this angle.
- replication-ondisk: 0 objection(s). The change only touches how the tablesync worker holds received COPY data in its own memory. It adds no on-disk, WAL, catalog, protocol or exported API change, so nothing crosses a process or version boundary. Without the code, I could not check whether the static copybuf state is reset if a worker ever syncs a second table.
- security: 0 objection(s). The patch changes only how copy_read_data() in tablesync.c keeps its pointer, length and cursor. It adds no privilege, role, setting, view, error text or user-reachable path, and the COPY data comes from the publisher as before. The tablesync worker's trust in that data does not change, so there is no security or privilege angle to raise.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection. All three versions change one file, add no hot-path cost, GUC or API, and v3 reuses initReadOnlyStringInfo(), which already has many callers in the tree, including worker.c and walreceiver.c. [P20] does not apply, because initReadOnlyStringInfo() was added for buffers that are not NUL-terminated. The thread still needs Álvaro Herrera's answer to v3. Without the code I could not check whether v3 still clears copybuf before each COPY, as v1 did.
- already-exists: 0 objection(s). This is an internal cleanup with no user-facing feature, so no existing feature or extension can replace it. v3 reuses initReadOnlyStringInfo() and drops v2's separate CopyBuf struct, which settles the duplicate-mechanism concern from message #3. [P20] concerned NUL termination, and initReadOnlyStringInfo() is meant for non-NUL-terminated buffers, so that precedent does not apply.
- demand: 0 objection(s). This is a refactoring-only cleanup, so a lack of user demand is not a reason to reject it. Its users are the people who maintain tablesync.c. Álvaro Herrera and Denis Smirnov both engaged on how the code should look, and the cost to everyone else is close to zero because nothing user-visible changes.

### Precedents retrieved

- P1: [Fix and improve allocation formulas](https://commitfest.postgresql.org/patch/6310/) (cf, Withdrawn, 2026-01-13) - judged not relevant; reason summarised from its thread
- P2: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P3: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant; reason summarised from its thread
- P5: [pre-allocating WAL files](https://commitfest.postgresql.org/patch/3158/) (cf, Withdrawn, 2022-07-25) - judged not relevant; reason summarised from its thread
- P6: [Add function to release an allocated SQLDA](https://commitfest.postgresql.org/patch/1669/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P7: [Optimize memory allocation in function "bringetbitmap"](https://commitfest.postgresql.org/patch/390/) (cf, Returned with feedback, 2015-12-23) - judged not relevant; reason summarised from its thread
- P8: [A infrastructure to detect misuse of spin lock easier](https://commitfest.postgresql.org/patch/4768/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P10: [Default to ICU during initdb](https://commitfest.postgresql.org/patch/4169/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P11: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P12: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P13: [pgbench - break out initialization timing info by init phase](https://commitfest.postgresql.org/patch/1493/) (cf, Returned with feedback, 2018-03-07) - judged not relevant
- P14: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P15: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant
- P16: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant
- P17: ["Causal reads" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/487/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P18: [Cygwin cleanup](https://commitfest.postgresql.org/patch/3781/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P19: [Revert "Permit dump/reload of not-too-large >1GB tuples"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b66adb7b0c83e632e0f881f828fa6f4233d01d06) (revert, Reverted, 2017-05-10) - judged not relevant
- P20: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - used in the review
- P21: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P22: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P23: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant
- P24: [share string to *int64 functions](https://commitfest.postgresql.org/patch/2099/) (cf, Returned with feedback, 2019-08-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 103 s; cost reported by the CLI: $1.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:02:44.373Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
