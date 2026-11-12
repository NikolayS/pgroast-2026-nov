# Design pre-review: EXPLAIN ANALYZE overcounts MERGE inserts suppressed by triggers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7606](https://commitfest.postgresql.org/patch/7606/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [EXPLAIN ANALYZE overcounts MERGE inserts suppressed by triggers](https://www.postgresql.org/message-id/flat/CAHewXNmZAG1i0PkRy%2Bb3koucghYn-ApBrYmtCgXDzYbRESA6Kg%40mail.gmail.com), started 2026-11-06
- Messages read: 1 of 1
- Patch prose read: 0001-Fix-MERGE-insert-counts-when-a-trigger-suppresses-in.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. Message #1 states the bug, gives a reproducer and explains why the caller cannot count correctly. The fix moves an existing counter and adds no user surface. Confidence is low because only one message is visible and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, three points are unchecked: the ON CONFLICT and plain INSERT paths through ExecInsert(), the count for FDW batch rows the remote side rejects, and the cross-partition UPDATE check.

## The proposal, restated

**Problem.** When a BEFORE INSERT trigger returns NULL for a MERGE insert, EXPLAIN ANALYZE still reports `Tuples Inserted: 1` and `Tuples Skipped: 0`, though the target table stays empty. Users who read MERGE statistics in EXPLAIN ANALYZE get wrong counts.

**Proposed design.** The patch moves the mt_merge_inserted increment from ExecMergeNotMatched() into ExecInsert(), after the row is inserted or queued for FDW batch insertion. The caller cannot tell from the returned NULL slot whether the insert happened. ExecInsert() checks the current MERGE action so the insert half of a cross-partition UPDATE counts only as an UPDATE.

**User-visible surface.**
- EXPLAIN ANALYZE output for MERGE: `Tuples Inserted` and `Tuples Skipped` change when a BEFORE INSERT trigger suppresses the row.

**What the author claims.**
- ExecInsert() returns NULL both for a trigger-suppressed insert and for a successful insert without RETURNING, so the caller cannot count correctly.
- ExecInsert() control flow already separates skipped rows from inserted or FDW-batched rows.
- Cross-partition UPDATE inside MERGE is still counted only as an UPDATE.
- The extended merge.sql trigger test checks the inserted and skipped counts.

**Thread state.** Only the first message is visible. Nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-06; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The entry is tagged Bugfix, and a back-patch would change EXPLAIN ANALYZE counts for MERGE in released branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | ExecInsert(), shared by all insert paths, gains a check of the current MERGE action. |
| performance on the hot path | **yes** | The patch adds a check in ExecInsert(), which every inserted row passes through. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches have the wrong count, and do you propose to back-patch the fix?
2. How does ExecInsert() tell a MERGE insert action from a plain INSERT, INSERT ... SELECT or ON CONFLICT call, so that those paths leave mt_merge_inserted unchanged?
3. Is a row queued for FDW batch insertion counted as inserted if the remote side later rejects or skips it?
4. Do MATCHED UPDATE and DELETE actions report wrong counts when a BEFORE trigger returns NULL?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the MERGE executor code in nodeModifyTable.c, including the cross-partition UPDATE and FDW batch insert paths, would have to own it. Nobody has replied yet.
- **Liability:** The owner signs for correct MERGE counts across all ExecInsert() callers, and for any change in EXPLAIN ANALYZE output on back branches if the fix is back-patched.
- **Author follow-up:** No sign either way; the thread has one message.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug affects users who read EXPLAIN ANALYZE counts for MERGE on tables with a BEFORE INSERT trigger that returns NULL. The patch is 4.3 KB, touches one executor file plus the merge regression test, and adds no user surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved list has no earlier attempt at this fix or a close relative, and message #1 points to no earlier discussion. From the prior-attempts angle there is no rejection reason to carry over.
- standards-compat: 0 objection(s). From the standards and compatibility angle, the only user-visible change is that EXPLAIN ANALYZE for MERGE reports `Tuples Inserted` and `Tuples Skipped` correctly when a BEFORE INSERT trigger suppresses a row. The patch adds no syntax, GUC, catalog object or psql change, and nothing in pg_dump is affected. A committer may still ask whether to back-patch, since tools or tests that saved the old counts would see them change.
- replication-ondisk: 0 objection(s). The patch changes only an in-memory executor counter that EXPLAIN ANALYZE shows. It does not touch on-disk format, WAL, replication, catalogs, the wire protocol or extension structs. A field mt_merge_inserted that already exists is updated in a different function, so nothing persists or crosses a process boundary.
- security: 0 objection(s). The patch only changes the MERGE insert and skip counters that EXPLAIN ANALYZE shows to the user who ran the statement. It adds no privilege, role, setting or new exposure, so there is no security angle to object on.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle, this is a narrow executor bugfix with no hot-path cost, catalog, GUC, API or external dependency. Moving the counter into ExecInsert() adds MERGE awareness to a shared function, but it fixes a wrong count without a new mechanism. Three points cannot be checked without the code: the ON CONFLICT paths, rows the remote side rejects after FDW batching, and back-patch scope.
- already-exists: 0 objection(s). This is a bug fix to EXPLAIN ANALYZE counters for MERGE. No existing feature, extension or setting corrects the wrong count. The patch changes the existing mt_merge_inserted counter and does not add a second mechanism, so the already-exists angle has nothing to raise.
- demand: 0 objection(s). This is a bug fix: EXPLAIN ANALYZE reports a wrong MERGE insert count, and message #1 gives a reproducer. The demand test does not apply to correcting existing output, and the fix adds no user surface for anyone else to pay for.

### Precedents retrieved

- P1: [Revert MERGE patch](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08ea7a2291db21a618d19d612c8060cda68f1892) (revert, Reverted, 2018-04-12) - judged not relevant
- P2: [MERGE](https://commitfest.postgresql.org/patch/1446/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P3: [explain analyze rows=%.0f](https://commitfest.postgresql.org/patch/3704/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P4: [Replace polyphase merge with a simple k-way balanced merge](https://commitfest.postgresql.org/patch/959/) (cf, Returned with feedback, 2017-02-27) - judged not relevant
- P5: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P6: [Sampling-based timing for EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/4100/) (cf, Returned with feedback, 2023-07-03) - judged not relevant; reason summarised from its thread
- P7: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [MERGE SQL command](https://commitfest.postgresql.org/patch/2920/) (cf, Returned with feedback, 2021-09-01) - judged not relevant; reason summarised from its thread
- P10: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant; reason summarised from its thread
- P11: [intXshr, intXshl: return error on shift count out of range](https://commitfest.postgresql.org/patch/7367/) (cf, Rejected, 2026-09-30) - judged not relevant; reason summarised from its thread
- P12: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant; reason summarised from its thread
- P13: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P14: [Support specify tablespace for each merged/split partition](https://commitfest.postgresql.org/patch/5157/) (cf, Rejected, 2024-10-08) - judged not relevant
- P15: [ci: Add MTEST_SUITES for optional test tailoring](https://commitfest.postgresql.org/patch/6430/) (cf, Withdrawn, 2026-06-05) - judged not relevant
- P16: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P17: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P18: [Revert "Add a test for transition table usage in FOR EACH ROW trigger."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a19ea9c6601bfb06dfd9f4c1060550dbc3f7bde1) (revert, Reverted, 2017-05-16) - judged not relevant
- P19: [suppressing useless wakeups in logical/worker.c](https://commitfest.postgresql.org/patch/4035/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P20: [suppress errors thrown by to_reg*()](https://commitfest.postgresql.org/patch/2060/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/4617/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P22: [Multi insert in CTAS/MatView](https://commitfest.postgresql.org/patch/2160/) (cf, Withdrawn, 2020-07-30) - judged not relevant
- P23: [Batched clock sweep to reduce cross-socket atomic contention](https://commitfest.postgresql.org/patch/7000/) (cf, Withdrawn, 2026-09-11) - judged not relevant
- P24: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 90 s; cost reported by the CLI: $0.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:43.476Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
