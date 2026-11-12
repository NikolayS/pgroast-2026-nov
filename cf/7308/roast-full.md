# Design pre-review: Optimize truncation logic to reduce AccessExclusive lock impact

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7308](https://commitfest.postgresql.org/patch/7308/) (Needs review); authors: shihao zhong (zhong950419), Stepan Neretin (snpgg)
- Thread: [Optimize truncation logic to reduce AccessExclusive lock impact](https://www.postgresql.org/message-id/flat/CA%2BYyo5Trs9x03-sc4PpmEXPY9K_B_jbEbNd01RbF%2BvOxs3zVUA%40mail.gmail.com), started 2025-03-18
- Messages read: 5 of 5
- Patch prose read: 03164ee7bce36946eaa521f6db2be1b30bead577.diff (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the lock problem is real and reported from production. Patch 2 makes the buffer drop on a primary depend on an exact fork size (prior-attempts-1, maintenance-fit-2), which is the doubt recorded for [P1]. Someone who knows the buffer manager has to judge that. Patch 1 still needs a workload that justifies it once patch 2 is applied (maintenance-fit-1).

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The material has no prose or file list for patch 2, so its API shape and release-build checks could not be judged. The safety argument is known only from the objections' account of message #5.

## The proposal, restated

**Problem.** When vacuum truncates free pages at the end of a table, it holds AccessExclusiveLock while smgrtruncate() scans all of shared buffers. With very large shared_buffers (the report cites 1.8 TiB on NUMA hosts), this scan is slow and blocks other sessions for a long time. pg_statistic is named as one of the most affected tables.

**Proposed design.** The thread has two patches. The first (Neretin, idea by Sokolov) changes should_attempt_truncation so vacuum truncates only when it can free at least 3% of the relation plus 2 pages. The second (Zhong, from Rowley's suggestion) has smgrtruncate() pass the known old fork sizes to DropRelationBuffers(), so small truncations look up buffers one by one instead of scanning the whole pool.

**User-visible surface.**
- Patch 1: vacuum truncates the relation tail less often, and small tables may stay at about 3 pages instead of 1.
- Patch 2: DropRelationBuffers() receives the old fork sizes, which is a C API change for extensions that call it.
- Patch 2: RelationTruncate() asserts that the caller holds AccessExclusiveLock, so extensions that call it with a weaker lock will hit the assertion.

**What the author claims.**
- Patch 2 cut DropRelationBuffers() time for a 2000-page truncation from 23.35 ms to 0.27 ms at shared_buffers 32GB.
- Below the 32768-block scan threshold at 8GB, a 30000-page truncation took about 2.5 to 3 times less time.
- Above the threshold both versions do the same full scan, and background throughput did not change in any run.
- Passed sizes are safe because AccessExclusiveLock or recovery rules stop the relation from growing between measuring and dropping.
- With zero_damaged_pages, a valid zero page past the end of file may stay for the checkpointer, which the author accepts.

**Thread state.** Rowley suggested the targeted-drop approach and noted that patch 1 only answers the existing comment about tuning REL_TRUNCATE_*. Sokolov agreed and asked whether huge tables should be truncated less often; patch 2 has no replies yet.

## 2. Prior attempts

### drop/truncate table sucks for large values of shared buffers (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/286/](https://commitfest.postgresql.org/patch/286/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)
- **Date:** decided 2015-07-02; started 2015-06-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Heikki Linnakangas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that the approach fails badly if the caller is wrong about the relation's maximum page number (for example, kernel lseek bugs). He also noted that it does not help DROP TABLESPACE or DROP DATABASE, which also scan the whole buffer pool. Heikki Linnakangas returned it with feedback, citing the issues raised, a still-unwritten alternative (delayed cleanup during the checkpoint scan), and a preference for a structure such as a radix tree that can find all buffers for a relation or database. Tom Lane added that committing a band-aid would make more complete fixes harder to pursue.
- **Reason (quote):** "Committing a band-aid will just make it harder to pursue real fixes." ([message](https://www.postgresql.org/message-id/14561.1435845813%40sss.pgh.pa.us))
- **What is different this time:** P1 aimed at DROP and TRUNCATE TABLE. Patch 2 covers only relation truncation and reuses the old_nblocks values that smgrtruncate() already has, with the existing FindAndDropRelationBuffers() path and threshold.


## 3. Objections a reviewer is likely to raise

### 1. Wrong relation size from lseek would leave buffers behind

fixable · level 2, rubric item 7 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

On a primary, vacuum truncates a table tail and smgrnblocks() returns a size that is too small, for example because of a kernel or file system bug. The targeted drop skips dirty buffers past that size. The checkpointer later tries to write a block of a truncated file and fails, and nobody sees it until then.

[P1] tried the same per-block lookup and, according to its thread summary, was turned down partly because it fails badly if the caller is wrong about the relation's last page. Message #5 argues only that the relation cannot grow between measuring and dropping. It does not cover a wrong size returned by lseek. The full scan does not depend on the reported size. d6ad34f34 limited the shortcut to recovery, where the size is cached, and this patch extends it to normal running.

**Precedents.**
- [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (Returned with feedback, 2015-07-02); thread: [link](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)

Thread: [#2 (David Rowley, 2025-03-19)](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#5 (shihao zhong, 2026-09-17)](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com)

**Question:** Outside recovery, what happens if smgrnblocks() underreports the fork size, and does the existing truncation path already fail in that case without the full scan?

Counter-check: held. Message #5 rules out relation growth but does not cover a size that smgrnblocks() underreports, which is the reason recorded for [P1]. The recovery path already trusts a size first read by lseek, which lowers the risk but does not answer the question.

### 2. Targeted drop makes truncate callers responsible for exact fork sizes

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A later patch or an extension calls smgrtruncate() with stale fork sizes, or while the relation can still grow. Buffers past the passed size stay in shared buffers, and the checkpointer later writes a dirty one past the new end of file. A release build gives no error at truncate time, so the problem shows up later as a checkpoint failure or a damaged relation.

Outside recovery, today's full scan does not depend on any size the caller passes. Message #5 moves correctness to the caller: the sizes must be exact and the relation must not grow, and only an Assert in RelationTruncate() enforces this. Outside recovery the size comes from lseek, and [P1] records doubt about trusting it as a reason the same lookup approach was not accepted. Reusing the existing FindAndDropRelationBuffers() path lowers the risk, but the proposal does not say what a release build checks.

**Precedents.**
- [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (Returned with feedback, 2015-07-02); thread: [link](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)

Thread: [#2 (David Rowley, 2025-03-19)](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#5 (shihao zhong, 2026-09-17)](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com)

**Question:** In a release build, what detects a dirty buffer left past the passed size, and why is the lseek-accuracy doubt recorded for the earlier attempt not a concern here?

Counter-check: held. smgrtruncate() callers already pass old_nblocks, but today a wrong value cannot leave buffers behind outside recovery. Patch 2 makes it a correctness input, and message #5 names only an Assert in RelationTruncate() as the check. The lseek part duplicates prior-attempts-1.

### 3. Patch 1 lets large tables keep much more empty tail space

fixable · level 1b, rubric item 4 · desirability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A user with a 2 TiB table deletes about 40 GiB of rows at its end. Today vacuum truncates once 1000 free tail pages (8 MiB) exist. Under 3% plus 2 pages it needs about 60 GiB, so the space never goes back to the OS, and the user finds out when disk usage does not drop.

Patch 1 changes a constant that applies to every installation, so all users of large tables pay in disk space to cut lock time on hosts with very large shared_buffers. Message #1 shows no data on how often truncation runs or how long the lock is held. Patch 2 already removes most of the scan cost for small truncations, which weakens the case for patch 1. Users who want fewer truncations can already set the vacuum_truncate reloption per table.

Thread: [#1 (Stepan Neretin, 2025-03-18)](https://www.postgresql.org/message-id/CA%2BYyo5Trs9x03-sc4PpmEXPY9K_B_jbEbNd01RbF%2BvOxs3zVUA%40mail.gmail.com), [#2 (David Rowley, 2025-03-19)](https://www.postgresql.org/message-id/CAApHDvrfngqCpQ09LKdr-BnJEVHW0%3DwAhxfRBnw%3DHJ2645cJPg%40mail.gmail.com), [#3 (Yura Sokolov, 2025-03-19)](https://www.postgresql.org/message-id/0be607a5-20f6-4894-bb22-412e3c784bc2%40postgrespro.ru), [#5 (shihao zhong, 2026-09-17)](https://www.postgresql.org/message-id/CAGRkXqRcNOrciQy%3D8sAYF%3D8Dsin2tYNpnUwe6V5JM%2BvXKeUHeA%40mail.gmail.com)

**Question:** With patch 2 applied, which workload still needs the 3% plus 2 pages rule, and what lock times show it?

Counter-check: held. The numbers are right: today 1000 free tail pages trigger truncation, and 3% of a 2 TiB table is about 60 GiB. Message #1 shows no lock-time data, and patch 2 makes Sokolov's small-truncation case cheap, so patch 1 still needs a stated workload.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Patch 2 changes which buffers are dropped at truncation, so a missed dirty buffer could be written past the new end of file. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 2 passes old fork sizes to DropRelationBuffers() and adds a lock assertion in RelationTruncate(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Patch 2 makes callers of smgrtruncate() responsible for exact fork sizes; patch 1 changes the default truncation rule for all tables. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Patch 1 overlaps the existing vacuum_truncate reloption and VACUUM TRUNCATE option. |

## 5. Questions the author must answer before anyone reviews code

1. Which patch is the CommitFest entry asking reviewers to look at: patch 2 alone, or both patches?
2. Outside recovery, what happens if smgrnblocks() underreports the fork size, and does the current truncation path already fail in that case without the full scan?
3. In a release build, what detects a dirty buffer left past the size passed to DropRelationBuffers()?
4. With patch 2 applied, which workload still needs the 3% plus 2 pages rule, and what lock times show it?
5. Does the lock problem remain for truncations above BUF_DROP_FULL_SCAN_THRESHOLD, and is that case in scope?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the buffer manager and smgr truncation code would have to own patch 2. Patch 2 has no replies yet.
- **Liability:** The owner signs for the rule that no dirty buffer survives past the new end of file after a targeted drop on a primary. A mistake shows up later as a checkpoint failure or a damaged relation, far from its cause.
- **Author follow-up:** The second author posted measurements and created the CommitFest entry. There is no sign either way about fixes after commit.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Patch 2 reuses an existing lookup path and helps installations with large shared_buffers, with measurements shown. Review cost is medium because the patch 2 code was not read and it touches the buffer manager; patch 1 can wait for a stated workload.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Patch 1 changes when every vacuum returns space, with no opt-in. Refuted (not specific): Duplicate of maintenance-fit-1, which uses the same 2 TiB example and the same vacuum_truncate alternative.
- `standards-compat-2` Patch 2 does not say whether DropRelationBuffers() keeps its signature. Refuted (no failure scenario): Postgres changes internal C functions between major versions, and a compile error is the loud signal extensions rely on. Truncating under a weaker lock is already unsafe, so the assertion exposes an existing bug in such a caller.
- `replication-ondisk-1` Targeted buffer drop trusts the measured file size outside recovery. Refuted (not specific): Duplicate of prior-attempts-1, which states the same lseek underreport scenario and asks the more precise question.
- `replication-ondisk-2` Valid buffer can survive past end of file after truncation. Refuted (no failure scenario): The same valid zero page past end of file exists today whenever no truncation follows the zero_damaged_pages read, so no invariant holds there now. Relation extension already handles an existing buffer for the new block and reuses a zero page.
- `already-exists-1` Patch 1 overlaps the existing vacuum_truncate storage parameter. Refuted (not specific): Duplicate of maintenance-fit-1, which already raises the vacuum_truncate reloption and the change of default for all tables.
- `demand-1` Patch 1 changes truncation for everyone to help huge buffer pools. Refuted (not specific): Duplicate of maintenance-fit-1. Its scenario is also partly wrong: for tables under about 33000 pages the current rule needs 1/16 of the table, so patch 1 truncates sooner there.

### Review angles

- prior-attempts: 1 objection(s). [P2] (prefetch in the vacuum backward scan) and [P5] (early lock release) are in the same area, but their recorded reasons do not apply to either patch.
- standards-compat: 2 objection(s). Neither patch adds SQL syntax, a GUC, a catalog change or pg_dump state. The objections cover the default behaviour change in patch 1 and the C API contract in patch 2.
- replication-ondisk: 2 objection(s). Patch 1 changes only the vacuum truncation threshold and has no persistent or cross-node state. Patch 2 does not change WAL or recovery behaviour, and the DropRelationBuffers() API change targets master only.
- security: 0 objection(s). Neither patch adds a privilege, role, setting, view or user-callable entry point, and neither changes what any session can see or do. The risk of missed buffers in patch 2 is a data safety question for the design angle, and patch 1 only changes when vacuum truncates.
- maintenance-fit: 2 objection(s). Two objections raised. Patch 2 reuses an existing lookup path, which lowers the maintenance concern, but no committer has replied to it yet.
- already-exists: 1 objection(s). Patch 2 extends the existing FindAndDropRelationBuffers() path used in recovery and does not add a second mechanism. No extension can change how smgrtruncate() drops buffers, so this angle raises nothing against patch 2.
- demand: 1 objection(s). The problem behind patch 2 is real and shown with a production report and measurements, and its benefit lands where the cost was, so the demand angle has no objection to patch 2.

### Precedents retrieved

- P1: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - used in the review; reason summarised from its thread
- P2: [optimizing vacuum truncation scans WIP](https://commitfest.postgresql.org/patch/220/) (cf, Returned with feedback, 2015-08-12) - used in the review; reason summarised from its thread
- P3: [Vacuuming relations on full tablespaces (Add EMERGENCY option)](https://commitfest.postgresql.org/patch/649/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P4: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P5: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - used in the review
- P6: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P7: [Allow shared buffer cache contents to affect index cost computations](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Update relfrozenxmin when truncating temp tables](https://commitfest.postgresql.org/patch/3358/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P11: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant; reason summarised from its thread
- P12: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant; reason summarised from its thread
- P13: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P14: [Optimize walsender handling invalid messages of 'drop publication'](https://commitfest.postgresql.org/patch/4393/) (cf, Withdrawn, 2023-10-20) - judged not relevant
- P15: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1366/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P16: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1421/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P17: [Add `truncate` option to subscription commands](https://commitfest.postgresql.org/patch/2758/) (cf, Withdrawn, 2020-11-25) - judged not relevant
- P18: [Rational number type (fractions)](https://commitfest.postgresql.org/patch/2441/) (cf, Rejected, 2020-09-04) - judged not relevant
- P19: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P20: [jit_warn_above_fraction parameter](https://commitfest.postgresql.org/patch/3573/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P22: [Store FullTransactionId in TwoPhaseFileHeader/GlobalTransactionData](https://commitfest.postgresql.org/patch/2226/) (cf, Withdrawn, 2019-09-04) - judged not relevant
- P23: [PGDLLIMPORT-ify old_snapshot_threshold.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=14216649f3dc8bd9839702440dd593e958b0920b) (revert, Partially reverted, 2016-04-21) - judged not relevant
- P24: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant

### Warnings

- review angle "replication-ondisk" returned prior attempts; ignored (only the prior-attempts angle may)
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 186 s; cost reported by the CLI: $1.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 93 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:39:07.369Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
