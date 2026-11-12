# Design pre-review: standby PANIC on restart after VM truncation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7386](https://commitfest.postgresql.org/patch/7386/) (Needs review); authors: shihao zhong (zhong950419), Rahul Yadav (rhyadav)
- Thread: [PostgreSQL 18.6/17.11: standby PANIC on restart after VM truncation](https://www.postgresql.org/message-id/flat/CAL4mQLAp562c1rCgg2Dqx6TBSdOk6vOvaLFjwFG9E6k4uwvJJw%40mail.gmail.com), started 2026-09-27
- Messages read: 11 of 11
- Patch prose read: v1-0001-Add-test-for-standby-restart-after-VM-truncation.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The thread states the problem with a reproducer and a field report, and three people confirmed it on four branches. v5 forgets invalid VM pages when truncation is replayed, as XLogTruncateRelation() already does for the main fork, so it follows existing practice. Confidence is medium because the material does not show v5's commit message, file list or code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The material shows only the v1 test patch, so v5's code, commit message and file list were not judged. Whether forgetting invalid VM pages at truncation replay could hide a real invalid-page reference cannot be checked without the code.

## The proposal, restated

**Problem.** A standby that already replayed a heap and VM truncation can PANIC with "WAL contains references to invalid pages" when it restarts recovery from an earlier restartpoint. The reporter says it reproduces on 18.6, 17.11, REL_18_STABLE and master, but not on 18.4 or 17.10. It hits after a switchover, and Shihao says also on a plain standby restart with full_page_writes = off.

**Proposed design.** Two fixes were posted. v4 reads VM pages in the VM-clear redo routines with RBM_ZERO_ON_ERROR and calls PageInit() on a page that comes back zeroed, so wal_consistency_checking still passes. Melanie's v5 keeps RBM_NORMAL and forgets invalid VM pages when truncation is replayed, the way the main fork already does through XLogTruncateRelation().

**What the author claims.**
- Kirill says the bug goes back to the first VM-clear redo commit, which lacked the invalid page guard that commit defe934 added.
- Rahul and Shihao report that v3 and v4 fix the PANIC on master, 19, 18 and 17, with and without wal_consistency_checking.
- Rahul found that RBM_ZERO_ON_ERROR alone fails under wal_consistency_checking = all, because heap_mask() rejects an all-zero page.
- Shihao says ignore_invalid_pages = on lets an affected standby start until the fix ships.
- Shihao reports that v5 lets the standby restart on master, 19, 18 and 17.

**Thread state.** Melanie posted v5 and prefers the truncation-replay fix, and Shihao accepted it and tested it on four branches. Open items are the v5 test, which does not apply on 17, and updating the CommitFest entry, which still points at the v4 approach.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Both fixes change how the startup process replays VM-clear and truncation records on a standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry targets stable branches, and the v5 test does not apply on 17. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fix adds invalid-page handling for the VM fork in redo code that needs care on every supported branch. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v5 pass with wal_consistency_checking = all on every branch it targets?
2. Does v5 cover the DELETE, HOT update and cross-page update cases that the v4 test checks with full_page_writes = off?
3. Which branches does v5 target, and which test goes into 17, where the v5 test does not apply?
4. Why do 18.4 and 17.10 pass the reproducer while 18.6 and 17.11 fail, if the bug dates from the first VM-clear redo commit?
5. Will the CommitFest entry be updated to point at v5 instead of the v4 approach?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Melanie Plageman wrote v5 and prefers the truncation-replay fix; the owner needs to know the visibility map redo code and invalid-page tracking in recovery.
- **Liability:** The owner signs for recovery correctness of VM truncation replay on master and every back branch that gets the fix, for the life of those branches.
- **Author follow-up:** Shihao accepted v5 and tested it on master, 19, 18 and 17; Rahul tested the earlier versions on the same branches.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** A standby PANIC on restart in released minor versions affects users of streaming replication, and v5 reuses the main-fork mechanism without new user-visible surface. Review cost is medium because no fix patch was read and the change is in recovery code that goes to stable branches.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and none of the retrieved candidates is the same or a similar idea. The thread points only to related work that is still open: commit defe934 (message #2), which added the main-fork invalid page guard that v5 now extends to the VM fork, and Melanie's VM clear thread (messages #3 to #5), which v5 replaces.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. Both v4 and v5 change only WAL redo behaviour for visibility map pages, with no new syntax, GUC, catalog entry, pg_dump output, psql change or client-visible behaviour. The backpatch to 17 changes no user-facing contract. The open items are choosing between v4 and v5, wal_consistency_checking coverage for v5, and a test for 17. These are design and test questions for other angles.
- replication-ondisk: 0 objection(s). This is a redo-only bug fix. It does not change the on-disk format, WAL record layout, catalogs, wire protocol or extension ABI, so pg_upgrade and mixed-version standbys are not affected. v5 forgets invalid VM pages when truncation is replayed, the same way XLogTruncateRelation() already does for the main fork, which follows existing practice. Questions only: why 18.4 and 17.10 pass, and whether v5 passes with wal_consistency_checking = all.
- security: 0 objection(s). Both fixes change only how the startup process replays visibility map pages during recovery, and they add no new privilege, role, setting or user-visible surface. Any user who can vacuum a table can trigger the PANIC today, and both fixes remove that; the ignore_invalid_pages workaround is already superuser-only.
- maintenance-fit: 0 objection(s). This is a back-patchable fix for a standby PANIC, and a committer who knows the visibility map code, Melanie Plageman, wrote v5 (message #10). v5 handles VM truncation replay the same way the main fork already does through XLogTruncateRelation(), so it adds no new mechanism, GUC, catalog change or hot-path cost for users who do not hit the bug. The open items are which test goes into 17 and whether v5 is checked under wal_consistency_checking = all and against the DELETE and update cases in the v4 test (messages #3, #11); those are test questions outside this angle.
- already-exists: 0 objection(s). This is a recovery bug fix, and no existing feature removes the PANIC. ignore_invalid_pages = on, named in message #3, is a temporary workaround that turns off a safety check, so it does not replace the fix. The thread has moved to v5, which reuses the main-fork forget-invalid-pages path in XLogTruncateRelation() and does not add a second mechanism.
- demand: 0 objection(s). This is a bug fix for a recovery PANIC in released minor versions. A field report on Linux, a reproducer, and independent confirmation on four branches by three people show the problem is real. A committer engaged and posted a fix. A demand objection does not apply to a crash on standby restart.

### Precedents retrieved

- P1: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant; reason summarised from its thread
- P2: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant; reason summarised from its thread
- P3: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P4: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P5: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P6: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P7: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant; reason summarised from its thread
- P8: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant; reason summarised from its thread
- P9: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P10: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P11: [Add pg_upgrade check for invalid role references in pg_init_privs](https://commitfest.postgresql.org/patch/6856/) (cf, Withdrawn, 2026-06-17) - judged not relevant; reason summarised from its thread
- P12: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P13: [Add a pg_truncate_freespace_map function](https://commitfest.postgresql.org/patch/4882/) (cf, Withdrawn, 2024-08-20) - judged not relevant
- P14: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P15: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - judged not relevant
- P16: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P17: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant
- P18: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant
- P19: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P20: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P21: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P22: [Panic during xlog building with big values](https://commitfest.postgresql.org/patch/5970/) (cf, Rejected, 2026-02-26) - judged not relevant
- P23: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P24: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 77 s; cost reported by the CLI: $0.83
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:49:00.062Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
