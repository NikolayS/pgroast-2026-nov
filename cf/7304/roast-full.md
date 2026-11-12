# Design pre-review: Discard aborted updaters when expanding a multixact

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7304](https://commitfest.postgresql.org/patch/7304/) (Needs review); authors: Chee Wooson (chee.wooson)
- Thread: [\[PATCH\] Discard aborted updaters when expanding a multixact](https://www.postgresql.org/message-id/flat/5778D6297D3CCF21%2B202609151836512678486%40vastdata.com.cn), started 2026-09-15
- Messages read: 9 of 9
- Patch prose read: v4-0001-Wait-for-updaters-still-in-progress-when-checking.patch, v4-0002-Judge-a-finished-multixact-updater-by-commit-stat.patch, v4-0003-Test-abort-window-conflicts-and-crashed-multixact.patch, v1-0001-Discard-aborted-updaters-in-MultiXactIdExpand.patch, v1-0002-Test-multixact-expansion-during-updater-abort.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The thread states the problem with a concrete interleaving, an independent pgbench reproducer on released versions (message #7) and a 2022 report with the same error. The fix changes existing conflict checks in heapam.c and adds no new user-visible surface. Confidence is medium because the code and the final commit were not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and the final commit were not read, so the claim that every caller of the changed conflict check tolerates the added wait is unverified. The remaining TransactionIdDidAbort() use in heap_lock_updated_tuple_rec() rests on the author's analysis in message #3.

## The proposal, restated

**Problem.** Concurrent updates fail with "new multixact has more than one updating member" when an updater has recorded its abort in pg_xact but is still in ProcArray. A crashed updater without an abort record also causes a spurious TM_Updated after recovery when a prepared key-share locker survives. Users with high-concurrency workloads that combine FK locks and rolled-back updates hit this.

**Proposed design.** The conflict check in heapam.c ignores a multixact updater only after it is no longer running and did not commit, so conflicting requests wait for the abort to finish. heap_update() then judges a finished updater by whether TransactionIdDidCommit() returns false instead of by TransactionIdDidAbort(). The v1 change to MultiXactIdExpand() was dropped in v3; v5 is Heikki's rewrite of the comments.

**User-visible surface.**
- New injection point in xact.c after the abort is written to pg_xact and before ProcArray cleanup (test use).

**What the author claims.**
- Removes the two-updater error without changing MultiXactIdExpand(); its existing filter discards the aborted member once the conflicting request has waited.
- Fixes a spurious serialization error at REPEATABLE READ and a tmfd.traversed assertion at READ COMMITTED for crashed updaters.
- heap_lock_updated_tuple_rec() can lock a dead successor and write extra WAL, but the author found no user-visible issue and left it unchanged.
- Lionel Hamayon reports 43 of 48 pgbench clients failing within 30 seconds on REL_18_STABLE, and no errors in about 16 million transactions with v5.
- Released versions 18.1 and 18.4 are affected; a 2022 report on 14.5 shows the same error.

**Thread state.** Heikki reviewed, rewrote the comments as v5, and reports he committed and back-patched it to all supported versions on 2026-10-15. The CommitFest entry still shows Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-15; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The thread reports a back-patch to all supported versions, and the tests add an injection point in xact.c. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes multixact updater checks in heapam.c, an area few people know well. |
| performance on the hot path | **yes** | Conflicting requests now wait until the aborting updater leaves ProcArray; the thread shows no measurement of that wait. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches received the injection point in xact.c and the new isolation and recovery tests?
2. Is there a measurement of the extra wait for conflicting requests between the pg_xact abort write and ProcArray cleanup under high concurrency?
3. Should heap_lock_updated_tuple_rec() also test that TransactionIdDidCommit() returns false, so that it does not lock a dead successor of a crashed updater?
4. Does the CommitFest entry need its status changed from Needs review to Committed?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas is reviewer and committer on the CommitFest entry, rewrote the comments as v5, and reports the commit and back-patch in the thread.
- **Liability:** The committer signs for the changed conflict and updater-status checks in heapam.c on all supported branches, including any later lock-wait or visibility reports from that code.
- **Author follow-up:** The author rebased twice, widened the fix in v3 after finding the crashed-updater case, and audited related TransactionIdDidAbort() calls.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bug hits workloads that mix foreign-key locks and rolled-back updates, with 43 of 48 pgbench clients failing in the report on REL_18_STABLE. The code patches are small and add no new surface, but the heapam.c multixact logic needs a reviewer who knows it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier rejected, returned or reverted attempt at this fix. Message #1 points to a 2022 report on 14.5 with the same error, and the thread presents it as a bug report that this patch resolves, so it gives no rejection reason to apply here. Heikki reviewed the patch and reports in message #9 that it was committed and back-patched.
- standards-compat: 0 objection(s). No standards or compatibility objection applies. The fix adds no syntax, GUC, catalog object, dump content or client-visible protocol change, and its only behaviour changes remove an error and a spurious serialization failure. The new injection point in xact.c is for tests only.
- replication-ondisk: 0 objection(s). The fix changes only in-memory conflict and status decisions in heapam.c, plus a test-only injection point in xact.c. It adds no on-disk format, WAL record, catalog, wire protocol or struct change, so pg_upgrade, standbys, logical decoding and back-branch ABI are not affected. The extra WAL from heap_lock_updated_tuple_rec() on a crashed creator's successor exists today and the patch leaves it as it is. Whether the recovery test with a prepared transaction reached the back branches cannot be checked from the material.
- security: 0 objection(s). The fix has no privilege or exposure aspect, since it only changes how heap_update and the multixact conflict check treat an aborted or crashed updater. The added wait lasts until ProcArray cleanup, which is a short window that a user cannot extend, and the new injection point in xact.c exists only in builds with injection points enabled.
- maintenance-fit: 0 objection(s). This is a correctness fix in heapam.c conflict checks, reviewed and committed by Heikki, an engaged committer who knows the area (message #9), with an independent plain-SQL reproducer (message #7). It adds no new mechanism, GUC or API, and the only added waiting is the short window between the pg_xact abort write and ProcArray cleanup. Two points could not be checked without the code or the final commit: which branches received the injection point and tests, and whether heap_lock_updated_tuple_rec() still uses TransactionIdDidAbort(). Message #3 says the author analyzed that function and found no user-visible issue.
- already-exists: 0 objection(s). This is a correctness fix inside heapam.c for a race in the multixact updater checks. No existing feature, setting, extension or documented workaround avoids the "new multixact has more than one updating member" error or the spurious TM_Updated for crashed updaters. The fix changes existing checks and adds no second mechanism, so the already-exists angle has nothing to object to.
- demand: 0 objection(s). The material shows real demand for this bug fix. There is an independent field report (message #7) with a plain-SQL pgbench reproducer that fails on released 18.1 and 18.4, the author hit it under TPC-C (message #8), and a 2022 report on 14.5 shows the same error (message #1). The fix changes only internal heapam.c behaviour and adds a test-only injection point, so no cost falls on non-users.

### Precedents retrieved

- P1: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P2: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P3: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P4: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P5: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P6: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P7: [Allow tuple visibility checks without hint-bit maintenance](https://commitfest.postgresql.org/patch/7212/) (cf, Withdrawn, 2026-08-30) - judged not relevant; reason summarised from its thread
- P8: [Adding a { LOCK ROWS | LOCK KEY INDEX } clause to FK definitions](https://commitfest.postgresql.org/patch/6923/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P9: [Allow tuple visibility checks without hint-bit, maintenance](https://commitfest.postgresql.org/patch/7215/) (cf, Withdrawn, 2026-08-29) - judged not relevant; reason summarised from its thread
- P10: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant; reason summarised from its thread
- P11: [Refactor how we form HeapTuples for CatalogTuple(Insert|Update)](https://commitfest.postgresql.org/patch/6221/) (cf, Withdrawn, 2026-01-30) - judged not relevant; reason summarised from its thread
- P12: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P13: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant
- P14: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P15: [Avoid hiding shared filesets in pg_ls_tmpdir (pg_ls_* functions for showing metadata ...)](https://commitfest.postgresql.org/patch/2377/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P16: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P17: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P18: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P19: [further improving roles_is_member_of()](https://commitfest.postgresql.org/patch/4925/) (cf, Withdrawn, 2024-04-13) - judged not relevant
- P20: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - judged not relevant
- P21: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P22: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant
- P23: [Speed up transaction completion faster after many relations are accessed in a transaction](https://commitfest.postgresql.org/patch/1993/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P24: [Pruning never visible changes](https://commitfest.postgresql.org/patch/3900/) (cf, Returned with feedback, 2022-10-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 77 s; cost reported by the CLI: $0.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:38:30.035Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
