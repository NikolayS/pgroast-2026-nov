# Design pre-review: Add recovery boundary WAL record for database and tablespace commands

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7272](https://commitfest.postgresql.org/patch/7272/); [thread](https://www.postgresql.org/message-id/flat/CACSdjfPmjJO1%3DLo9qbCmyc9SBKmJQ4YV_mFbCPkXZf7g5%2BQdeg%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has shown demand and no objection at level 1 survived. The design does not yet meet its own aim: replication-ondisk-1 shows a time target between the boundary record and the commit still leaves a broken cluster, and standards-compat-1 shows recovery_target_inclusive changes the result at an equal timestamp. The list needs to agree whether a narrowed window is acceptable before code review. maintenance-fit-1 is a smaller design point.

Not checked: The code was not read. The handling of the new record in decode.c, the exact placement of the boundary record at each call site, and the interaction of the two waits in 0002 could not be judged.

## Prior attempts

- [PITR enhance getRecordTimestamp()](https://www.postgresql.org/message-id/flat/CANbhV-F+8=Y=cfurfD2hjoWVUvTk-Ot9BJdw2Myc=st3TsZy9g@mail.gmail.com) (similar idea; Withdrawn, 2022-09-16). reason not recorded Different now: The row's title suggests it changed getRecordTimestamp() so recovery could read times from more existing records. This proposal adds a new timestamped record for five database and tablespace commands. The row records no reason for the withdrawal.

## Objections a reviewer is likely to raise

### 1. Boundary record leaves a window where PITR still breaks the catalog

An operator sets recovery_target_time to a moment while DROP DATABASE or CREATE DATABASE was waiting on its checkpoints. Recovery passes the boundary record, replays the file operation, then stops before the commit. After promotion the operator finds a database that cannot be connected to, or an orphan base/<oid>.

The invariant the patch aims for is that the catalog and the directories agree after a time-targeted PITR. Message #4 shows this still fails for any target between the boundary and the commit, and these commands wait on checkpoints in that span. The proposal only narrows the window and does not say so. The author has not replied, and message #4 also found an inclusive = on equality case where the comparison lets the file operation through.

**Question:** Is a partial fix acceptable for the window between boundary and commit, and if so, how will operators learn which targets remain unsafe?

<sub>fixable, level 2, WAL / replication / standby; messages [#3](https://www.postgresql.org/message-id/CACSdjfNo1CVKDTBRbuuZipv-4s2ozirv1Hjpace%2Bq0iQfGiuug%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Inclusive and exclusive time targets give different results at the boundary

An operator sets recovery_target_time to a time recorded just before a DROP DATABASE and leaves recovery_target_inclusive = on. Recovery replays the directory removal and stops before the commit, and connecting fails with "cannot connect to invalid database". The same target with inclusive = off leaves a clean cluster.

0001 reuses the commit-record comparison for a record that is not a transaction and cannot be "included", so recovery_target_inclusive has no clear meaning for it. Message #4 shows the two settings giving different results for the same WAL and suggests >= for both. 0001 adds no documentation of the new stop point, and the author has not replied to message #4.

**Question:** What should recovery_target_inclusive mean for a boundary record, and where do the recovery_target_time docs state the new stop point and its limits?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Every future file-operating DDL path must remember the boundary record

A later patch adds a new CREATE DATABASE strategy or a new directory operation in dbcommands.c or tablespace.c without emitting the boundary record first. No test or assertion fails. Operators find out after a PITR to a time target leaves an orphan or missing directory, the same bug this patch fixes.

The fix relies on placing a call before "the first irreversible record" at each of five call sites in two files. Nothing in recovery checks that a non-transactional file record follows a boundary record. The new tests in 003_recovery_targets.pl cover only today's five commands, so a reviewer of a later patch is likely to miss the requirement.

**Question:** What makes a future file-operating record without a preceding boundary record fail loudly, for example an assertion or a check during replay?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CACSdjfPmjJO1%3DLo9qbCmyc9SBKmJQ4YV_mFbCPkXZf7g5%2BQdeg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CADMs5OMeBCK6rNKVD3BdM6g%3Dia_4MdrdDoH5GiYdhi6%2B3bSTjA%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Is a partial fix intended for time targets between the boundary record and the commit, and if so, how do operators learn which targets remain unsafe?
2. What should recovery_target_inclusive mean for a boundary record, and should the comparison be >= for both settings as message #4 suggests?
3. What makes a future file-operating WAL record without a preceding boundary record fail loudly, for example an assertion or a check during replay?
4. Where will the recovery_target_time documentation describe the new stop point, given that 0001 has no documentation change?
5. Why does 0001 touch decode.c, and what does logical decoding do with the new record?

## Touch points

- **on-disk format / pg_upgrade:** 0001 bumps XLOG_PAGE_MAGIC and touches pg_control.h, so the WAL format on disk changes.
- **WAL / replication / standby:** 0001 adds the XLOG2 RECOVERY_BOUNDARY record and a new stop point; 0002 changes what a delayed standby waits on.
- **backpatchability:** The entry is tagged Bugfix, but the new record type and XLOG_PAGE_MAGIC bump limit it to version 20.
- **maintenance burden ("who maintains this in 10 years"):** Five call sites in dbcommands.c and tablespace.c must each write the boundary record, and future file-operating commands must do the same.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the recovery target and replay delay code in xlogrecovery.c and the WAL logging of database and tablespace commands.
- Priority: fix design first; audience a sizeable minority; review cost medium. Two field reports and a tester's reproduction show real demand among users of time-targeted PITR and delayed standbys. The patch is small, but it touches recovery stop logic, and the window and recovery_target_inclusive questions from message #4 are open.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
