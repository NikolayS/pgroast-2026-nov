# Design pre-review: Add recovery boundary WAL record for database and tablespace commands

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7272](https://commitfest.postgresql.org/patch/7272/) (Needs review); authors: JoongHyuk Shin (shin)
- Thread: [\[PATCH\] Add recovery boundary WAL record for database and tablespace commands](https://www.postgresql.org/message-id/flat/CACSdjfPmjJO1%3DLo9qbCmyc9SBKmJQ4YV_mFbCPkXZf7g5%2BQdeg%40mail.gmail.com), started 2026-09-07
- Messages read: 4 of 4
- Patch prose read: v2-0001-Add-recovery-boundary-WAL-record-for-database-and.patch, v2-0002-Make-recovery_min_apply_delay-wait-on-recovery-bo.patch, 0002-Make-recovery_min_apply_delay-wait-on-recovery-bound.patch, 0001-Add-recovery-boundary-WAL-record-for-database-and-ta.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has shown demand and no objection at level 1 survived. The design does not yet meet its own aim: replication-ondisk-1 shows a time target between the boundary record and the commit still leaves a broken cluster, and standards-compat-1 shows recovery_target_inclusive changes the result at an equal timestamp. The list needs to agree whether a narrowed window is acceptable before code review. maintenance-fit-1 is a smaller design point.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The handling of the new record in decode.c, the exact placement of the boundary record at each call site, and the interaction of the two waits in 0002 could not be judged.

## The proposal, restated

**Problem.** recovery_target_time is checked only on commit and abort records, so a time target set before CREATE/DROP DATABASE, ALTER DATABASE SET TABLESPACE or CREATE/DROP TABLESPACE does not stop recovery before their file operations. After PITR the cluster has a catalog entry with no directory, or an orphan directory. Delayed standbys also apply these file operations at once, ignoring recovery_min_apply_delay.

**Proposed design.** 0001 writes a new XLOG2 record with a timestamp before the first irreversible record of each of the five commands, and recoveryStopsBefore() stops before it when the timestamp is past a recovery_target_time target. The record has no redo action and does not affect other target types. 0002 makes recovery_min_apply_delay also wait on this record.

**User-visible surface.**
- New XLOG2 WAL record type RECOVERY_BOUNDARY with a timestamp, shown by pg_waldump; XLOG_PAGE_MAGIC is bumped.
- New log message "recovery stopping before recovery boundary, time ...".
- recovery_target_time can now stop recovery before these five commands, not only at commit or abort.
- recovery_min_apply_delay now also delays the file operations of these commands; the config.sgml paragraph changes.

**What the author claims.**
- Stopping before the boundary record loses nothing, because replaying the record has no effect.
- The bug was reported twice before, in 2019 (loxodata) and later (pgbackrest).
- xid and LSN targets with inclusive = off show the same result on master; the author treats that as a separate, documented behaviour.
- v2 is only a rebase of v1.

**Thread state.** Shihao Zhong asked about xid targets; the author declined to cover them. João Rito tested v2 and confirmed the fix, but reported that targets between the boundary and the commit still break the database, an inclusive = on equality edge case, and missing docs in 0001; the author has not replied.

## 2. Prior attempts

### PITR enhance getRecordTimestamp() (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3222/](https://commitfest.postgresql.org/patch/3222/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANbhV-F+8=Y=cfurfD2hjoWVUvTk-Ot9BJdw2Myc=st3TsZy9g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANbhV-F+8=Y=cfurfD2hjoWVUvTk-Ot9BJdw2Myc=st3TsZy9g@mail.gmail.com)
- **Date:** decided 2022-09-16; started 2021-06-30
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** The row's title suggests it changed getRecordTimestamp() so recovery could read times from more existing records. This proposal adds a new timestamped record for five database and tablespace commands. The row records no reason for the withdrawal.


## 3. Objections a reviewer is likely to raise

### 1. Boundary record leaves a window where PITR still breaks the catalog

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

An operator sets recovery_target_time to a moment while DROP DATABASE or CREATE DATABASE was waiting on its checkpoints. Recovery passes the boundary record, replays the file operation, then stops before the commit. After promotion the operator finds a database that cannot be connected to, or an orphan base/<oid>.

The invariant the patch aims for is that the catalog and the directories agree after a time-targeted PITR. Message #4 shows this still fails for any target between the boundary and the commit, and these commands wait on checkpoints in that span. The proposal only narrows the window and does not say so. The author has not replied, and message #4 also found an inclusive = on equality case where the comparison lets the file operation through.

Thread: [#3 (JoongHyuk Shin, 2026-11-05)](https://www.postgresql.org/message-id/CACSdjfNo1CVKDTBRbuuZipv-4s2ozirv1Hjpace%2Bq0iQfGiuug%40mail.gmail.com), [#4 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com)

**Question:** Is a partial fix acceptable for the window between boundary and commit, and if so, how will operators learn which targets remain unsafe?

Counter-check: held. Message #4 reproduced a broken cluster for targets between the boundary record and the commit, with a window of about one second for CREATE DATABASE. The proposal does not state this limit, and the author has not answered.

### 2. Inclusive and exclusive time targets give different results at the boundary

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

An operator sets recovery_target_time to a time recorded just before a DROP DATABASE and leaves recovery_target_inclusive = on. Recovery replays the directory removal and stops before the commit, and connecting fails with "cannot connect to invalid database". The same target with inclusive = off leaves a clean cluster.

0001 reuses the commit-record comparison for a record that is not a transaction and cannot be "included", so recovery_target_inclusive has no clear meaning for it. Message #4 shows the two settings giving different results for the same WAL and suggests >= for both. 0001 adds no documentation of the new stop point, and the author has not replied to message #4.

Thread: [#4 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com)

**Question:** What should recovery_target_inclusive mean for a boundary record, and where do the recovery_target_time docs state the new stop point and its limits?

Counter-check: held. Message #4 shows inclusive = on and off give different results when the target equals the boundary time, and the author has not replied. The scenario should say "exactly equal": targets just before the command were clean for both settings.

### 3. Every future file-operating DDL path must remember the boundary record

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A later patch adds a new CREATE DATABASE strategy or a new directory operation in dbcommands.c or tablespace.c without emitting the boundary record first. No test or assertion fails. Operators find out after a PITR to a time target leaves an orphan or missing directory, the same bug this patch fixes.

The fix relies on placing a call before "the first irreversible record" at each of five call sites in two files. Nothing in recovery checks that a non-transactional file record follows a boundary record. The new tests in 003_recovery_targets.pl cover only today's five commands, so a reviewer of a later patch is likely to miss the requirement.

Thread: [#1 (JoongHyuk Shin, 2026-09-07)](https://www.postgresql.org/message-id/CACSdjfPmjJO1%3DLo9qbCmyc9SBKmJQ4YV_mFbCPkXZf7g5%2BQdeg%40mail.gmail.com), [#4 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com)

**Question:** What makes a future file-operating record without a preceding boundary record fail loudly, for example an assertion or a check during replay?

Counter-check: held. The fix depends on a call placed by hand at five sites, and the material shows no replay check or assertion that catches a missing one. A missed site only returns to today's behaviour and such commands are rare, so fixable is the right severity.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | 0001 bumps XLOG_PAGE_MAGIC and touches pg_control.h, so the WAL format on disk changes. |
| WAL / replication / standby | **yes** | 0001 adds the XLOG2 RECOVERY_BOUNDARY record and a new stop point; 0002 changes what a delayed standby waits on. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but the new record type and XLOG_PAGE_MAGIC bump limit it to version 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Five call sites in dbcommands.c and tablespace.c must each write the boundary record, and future file-operating commands must do the same. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is a partial fix intended for time targets between the boundary record and the commit, and if so, how do operators learn which targets remain unsafe?
2. What should recovery_target_inclusive mean for a boundary record, and should the comparison be >= for both settings as message #4 suggests?
3. What makes a future file-operating WAL record without a preceding boundary record fail loudly, for example an assertion or a check during replay?
4. Where will the recovery_target_time documentation describe the new stop point, given that 0001 has no documentation change?
5. Why does 0001 touch decode.c, and what does logical decoding do with the new record?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the recovery target and replay delay code in xlogrecovery.c and the WAL logging of database and tablespace commands.
- **Liability:** The owner signs for a new WAL record type, a new recovery stop point and a changed recovery_min_apply_delay rule. PITR results that still break inside the boundary-to-commit window would be reported against this commit.
- **Author follow-up:** The author answered message #2 within a day and posted a rebase. The author has not yet replied to message #4.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Two field reports and a tester's reproduction show real demand among users of time-targeted PITR and delayed standbys. The patch is small, but it touches recovery stop logic, and the window and recovery_target_inclusive questions from message #4 are open.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Bugfix tag conflicts with a WAL format change that cannot be back-patched. Refuted (already handled): The CommitFest entry targets version 20 and nobody in the thread proposes a back-patch, so the mixed minor release scenario does not arise. Back branches keeping an existing bug is the state today and is not a flaw in this design.
- `replication-ondisk-2` New WAL record and page magic bump conflict with the Bugfix tag. Refuted (not specific): Duplicate of standards-compat-2, which is refuted because the entry targets version 20 and no back-patch is proposed.
- `maintenance-fit-2` Permanent WAL record type for a fix that leaves part of the window open. Refuted (wrong facts): WAL is not carried across major versions, so a later major release can remove the record type, and the suggested check on file-operation records would still replay them for a target inside the window. The window itself is a duplicate of replication-ondisk-1.
- `maintenance-fit-3` Bugfix tag conflicts with a new WAL record and XLOG_PAGE_MAGIC bump. Refuted (not specific): Duplicate of standards-compat-2, which is refuted because the entry targets version 20 and no back-patch is proposed.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a reason that applies here. [P15] is the closest match but records no reason for its withdrawal. [P12] reverted a fix for replaying create database records on a standby, and its recorded reason concerns that specific approach. Message #1 cites two earlier reports of the bug, and message #2 says both have been open for years. Those are bug reports, not rejected fixes, so they support demand and give no grounds for rejection.
- standards-compat: 2 objection(s). Other contract points look settled. The xid and LSN behaviour is the existing documented behaviour (message #3), and 0002 updates the recovery_min_apply_delay docs for its behaviour change.
- replication-ondisk: 2 objection(s). Logical decoding of the new record (decode.c) and the double wait in 0002 could not be judged without the code. Neither yields a concrete failure scenario from the material.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The boundary record is written only by commands that already need CREATEDB, ownership or superuser, and adds no new data beyond a timestamp. recovery_target_time and recovery_min_apply_delay stay superuser-set, and 0002's delay matches the per-commit delay any user can already cause, so it adds no new denial-of-service path.
- maintenance-fit: 3 objection(s). Hot-path cost looks negligible because the record is written only by rare DDL. No new GUC, catalog or external dependency is involved.
- already-exists: 0 objection(s). No existing feature does this job. An operator can find the LSN of the first database or tablespace record with pg_waldump and recover with recovery_target_lsn, but only after learning that a time target silently produced a broken cluster, and no extension can add a timestamped WAL record to these commands. A committer is unlikely to call this a duplicate, because the record would give recovery its only timestamp before these file operations.
- demand: 0 objection(s). Demand is shown. Two independent field reports, from 2019 and later, are cited in message #1, and message #2 notes both stayed open for years. In message #4 a tester reproduced a missing database directory and orphan directories after PITR on master. A delayed standby that removes a dropped database's directory at once defeats the usual reason to run one, so 0002 also has a clear user group. The cost falls mostly on the five database and tablespace commands and on recovery_target_time users.

### Precedents retrieved

- P1: [Improve recovery performance of dropping database having many tablespaces](https://commitfest.postgresql.org/patch/1658/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P2: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P3: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P4: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P5: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P6: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2935/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P8: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P9: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P10: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P11: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P12: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - used in the review
- P13: [Add recovery_min_apply_delay_reconnect recovery option](https://commitfest.postgresql.org/patch/1325/) (cf, Returned with feedback, 2017-11-15) - judged not relevant
- P14: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P15: [PITR enhance getRecordTimestamp()](https://commitfest.postgresql.org/patch/3222/) (cf, Withdrawn, 2022-09-16) - used in the review
- P16: [using an end-of-recovery record in all cases](https://commitfest.postgresql.org/patch/3365/) (cf, Withdrawn, 2022-01-17) - judged not relevant
- P17: [orphaned function](https://commitfest.postgresql.org/patch/2850/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P18: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P19: [PITR shutdown should not report error by pg_ctl](https://commitfest.postgresql.org/patch/4653/) (cf, Withdrawn, 2023-11-30) - judged not relevant
- P20: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - judged not relevant
- P21: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant
- P22: [Avoid orphaned dependencies](https://commitfest.postgresql.org/patch/3106/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P23: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P24: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 158 s; cost reported by the CLI: $1.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 82 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:35:08.696Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
