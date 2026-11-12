# Design pre-review: Fix pg_combinebackup with truncated intermediate incremental backups

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7573](https://commitfest.postgresql.org/patch/7573/) (Needs review); authors: Rajesh Kumar Thandapani (rajeshkumart)
- Thread: [Re: BUG #19776: pg_combinebackup ignores the truncation_block_length of an intermediate incremental backup](https://www.postgresql.org/message-id/flat/CAJoZ81s07r3akkKp-nJ8sgQNoCM0nu24g3HZ0FxTSY1OqsgurQ%40mail.gmail.com), started 2026-10-28
- Messages read: 5 of 5
- Patch prose read: v3-0001-Fix-pg_combinebackup-with-truncated-intermediate-.patch, v1-0001-Fix-pg_combinebackup-with-truncated-intermediate-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The thread states the problem with a bug report and reproducer (BUG #19776), the reviewer in message #3 confirmed the failure on master and the pass with v2, and the fix adds no user-visible surface. The remaining gaps are about tests and branch choice, which a committer can settle.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the logic of the running bound and of the full_copy_possible line rests on the reviewer's account in message #3. The segment-boundary case on --with-segsize-blocks builds has no test.

## The proposal, restated

**Problem.** pg_combinebackup can restore stale rows when an intermediate incremental backup truncated a relation and a later backup extended it again without writing every block in that range, for example after COPY bulk extension. reconstruct_from_incremental_file() checks only the latest backup's truncation_block_length, so the unwritten blocks are copied from a backup taken before the truncation. Anyone who restores from a chain of incremental backups is affected.

**Proposed design.** While reconstruct_from_incremental_file() walks backward through the chain, it tracks a running truncation_block_length upper bound and lowers it whenever an incremental backup has a smaller value. Unassigned blocks at or above that bound stay unassigned and are written as zeroes. The bound applies after the current file's own blocks are taken, applies only to incremental files, and also clears full_copy_possible.

**What the author claims.**
- Without the fix, pg_combinebackup sources unwritten blocks from a pre-truncation backup and brings back stale rows.
- Unassigned blocks past the tightened bound are written as zeroes.
- v2 is a rebase of v1 with no functional changes.
- The extended 011_ib_truncation.pl covers an intermediate truncation followed by COPY bulk extension.

**Thread state.** One reviewer, Raja Sai Pranav, tested v2 on master (the new test fails without the fix and passes with it), asked for a clearer comment, and gave LGTM on v3 without re-running the TAP suite on v3. Fujii Masao is listed as a CommitFest reviewer but has not posted in the visible thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The fix changes which blocks pg_combinebackup writes into the reconstructed relation files; the incremental file format stays the same. |
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
| backpatchability | **yes** | This is a data-correctness bug fix, and the thread does not say which branches get it. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches need this fix, and does the patch apply to each of them?
2. Can the test be made to fail without the fix on a --with-segsize-blocks=6 build, for example with a truncation point inside the re-extended segment?
3. Can any test, perhaps one that builds backup files by hand, cover the full_copy_possible = false line?
4. Does the pg_combinebackup TAP suite pass on v3, given that the reported pass result is from v2?
5. What should users do who already restored from an affected chain and may have stale rows?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the incremental backup and pg_combinebackup reconstruction code. Fujii Masao is listed as a CommitFest reviewer but has not posted in the visible thread.
- **Liability:** The owner signs for the correctness of every backup that pg_combinebackup reconstructs from an incremental chain, on every branch that gets the fix. A mistake in the new bound would write zeroes over valid blocks.
- **Author follow-up:** The author posted a rebase in message #2 and a v3 after the review in message #3, which shows active follow-up within the first two weeks.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug restores stale rows for users of incremental backup chains, and the fix touches one source file and one test file with no new user-visible surface. A reviewer already confirmed the failure and the fix on master.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier proposal for the same or a similar idea, and the thread does not point to an earlier discussion. This is a bug fix for data restored from an incremental backup chain, so no prior rejection reason applies from this angle.
- standards-compat: 0 objection(s). From the standards and compatibility angle, this bug fix in reconstruct_from_incremental_file() gives no grounds for rejection. It adds no syntax, keyword, GUC, catalog object, psql behaviour or dump format. It uses the existing truncation_block_length field in incremental files, so backups taken before the fix stay valid input. The only behaviour change is that pg_combinebackup no longer restores truncated, stale blocks and writes zeroes instead, which is the intended correct result. The material does not yet say which branches get the fix, but a committer can settle that at commit time.
- replication-ondisk: 0 objection(s). No objection from the replication and on-disk angle. The patch changes how pg_combinebackup picks the source backup for each block. It does not change the incremental file format, WAL, the catalog, the wire protocol or any extension API, so it adds no pg_upgrade or rolling-upgrade concern. The design writes zeroes for blocks at or above the lowest truncation_block_length seen in the chain, unless a later backup supplied them. This keeps the existing invariant that a combined backup matches the latest backup state, and the reviewer in message #3 checked that reasoning. Several points could not be checked without the code. The segment-boundary case on --with-segsize-blocks builds has no test, and neither does the full_copy_possible line. The material also does not say which branches get the fix. Users who already restored from an affected chain have no way to detect stale rows, but that is a reason to fix the bug, not an objection to this design.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how pg_combinebackup, a client tool run by the backup operator on files that operator already owns, picks block sources, and it adds no SQL surface, privilege, role or setting. Resurrected rows are a data correctness problem, and the fix narrows what gets restored, so it gives no new access.
- maintenance-fit: 0 objection(s). This is a contained correctness fix inside pg_combinebackup that adds no API, GUC or catalog state and costs nothing for non-users, so there is no maintenance or fit objection. The remaining gaps are level 4 or committability items: the new test does not detect the bug on a --with-segsize-blocks=6 build, the full_copy_possible line has no test, v3 has not been re-run, and Fujii Masao has not posted.
- already-exists: 0 objection(s). This is a correctness fix inside pg_combinebackup's reconstruction logic. No existing feature, option or external tool gives a correct combined backup when an intermediate incremental backup truncated a relation, so the already-exists angle has nothing to raise.
- demand: 0 objection(s). This is a data-correctness bug fix for pg_combinebackup, reported by a user as BUG #19776 with a reproducer. Anyone restoring from an incremental chain can get stale rows back, so demand is real, and only the restore path pays the cost.

### Precedents retrieved

- P1: [teach pg_combinebackup to detect deleted files where possible](https://commitfest.postgresql.org/patch/4951/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P2: [File level incremental backup](https://commitfest.postgresql.org/patch/94/) (cf, Rejected, 2015-03-18) - judged not relevant; reason summarised from its thread
- P3: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P4: [Incremental sort](https://commitfest.postgresql.org/patch/1011/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P5: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [pg_get_domain_ddl: DDL reconstruction function for CREATE DOMAIN statement](https://commitfest.postgresql.org/patch/6983/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P7: [Move backup-related code to xlogbackup.c/.h](https://commitfest.postgresql.org/patch/3919/) (cf, Rejected, 2023-03-24) - judged not relevant; reason summarised from its thread
- P8: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Vacuuming relations on full tablespaces (Add EMERGENCY option)](https://commitfest.postgresql.org/patch/649/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P10: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant; reason summarised from its thread
- P11: [Revert "Get rid of WALBufMappingLock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c13070a27b63d9ce4850d88a63bf889a6fde26f0) (revert, Reverted, 2025-08-22) - judged not relevant
- P12: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant
- P14: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P15: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant
- P16: [DROP relation IF EXISTS Docs and Tests - Bug Fix](https://commitfest.postgresql.org/patch/2603/) (cf, Withdrawn, 2021-08-11) - judged not relevant
- P17: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P18: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant
- P19: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P20: ["Causal reads" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/487/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P21: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant
- P22: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant
- P23: [Variable Block Size](https://commitfest.postgresql.org/patch/4429/) (cf, Withdrawn, 2024-01-19) - judged not relevant
- P24: [Support to COMMENT ON DATABASE CURRENT_DATABASE](https://commitfest.postgresql.org/patch/1320/) (cf, Rejected, 2018-03-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 133 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:42.030Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
