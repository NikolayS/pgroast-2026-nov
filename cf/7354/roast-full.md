# Design pre-review: Use instr_time for pg_stat_database block read/write time counters

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7354](https://commitfest.postgresql.org/patch/7354/) (Needs review); authors: Bernd Reiß (berndreiss), Ahmed Gouda (gouda0x.gmail.com)
- Thread: [Use instr_time for pg_stat_database block read/write time counters](https://www.postgresql.org/message-id/flat/605732d2-c6bd-4c65-ac83-8d071935d860%40gmx.at), started 2026-09-28
- Messages read: 13 of 13
- Patch prose read: v1-0001-use-instr_time-for-pg_stat_database-io-time-counters.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible script in message #1, the change follows the instr_time practice that pg_stat_io and pgBufferUsage already use, and it adds no SQL, GUC, catalog or on-disk surface. The open point is sequencing against the planned pgBufferUsage rework, which the thread treats as separate later work.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody here could confirm that PgStat_StatDBEntry and the stats file keep microsecond PgStat_Counter values, or that every flush path converts the accumulated ticks exactly once. Only 3 of 13 messages were visible.

## The proposal, restated

**Problem.** pg_stat_database truncates each I/O duration to microseconds before it adds the duration to blk_read_time and blk_write_time, so the totals undercount and drift from pg_stat_io. Users who compare the two views, or who run fast storage with sub-microsecond I/O, see different numbers for the same work.

**Proposed design.** The patch changes pgStatBlockReadTime and pgStatBlockWriteTime from PgStat_Counter to instr_time, so the backend accumulates ticks and converts to microseconds only at stats flush. It removes the pgstat_count_buffer_{read|write}_time macros and makes the vacuum and analyze snapshot variables instr_time, which converts elapsed time straight to milliseconds.

**User-visible surface.**
- pg_stat_database.blk_read_time and blk_write_time report higher totals that match pg_stat_io more closely; the column types do not change.
- C API: the patch removes the pgstat_count_buffer_read_time and pgstat_count_buffer_write_time macros from pgstat.h.
- C API: pgStatBlockReadTime and pgStatBlockWriteTime change type from PgStat_Counter to instr_time.

**What the author claims.**
- With the patch, the pg_stat_database and pg_stat_io read/write totals almost match, where before write time differed by about 15%.
- Removing the per-I/O microsecond conversion from the hot path gives about 0.5% more tps in a single-client, I/O-heavy pgbench -S run.
- The on-disk stats format, catalogs and the vacuum log line format do not change (reviewer claim).
- The style now matches pg_stat_io, pgBufferUsage and function stats, which already accumulate in instr_time.

**Thread state.** Narayanan reviewed it and has no further code comments; Andres expects a measurable win but wants pg_stat_database to derive its times from pgBufferUsage later, in separate work. Lukas suggests doing that conversion after the parallel-worker double-counting fix is settled in the WAL statistics thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pgstat.h loses pgstat_count_buffer_{read\|write}_time, and pgStatBlockReadTime and pgStatBlockWriteTime change type to instr_time. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The header change and the step in reported blk_read_time and blk_write_time values fit a major release only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The planned pgBufferUsage rework would later replace the counters this patch retypes. |
| performance on the hot path | **yes** | The patch changes pgstat_count_io_op_time(), which runs on every timed read, write and extend. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should this patch go in before the planned pgBufferUsage-based rework of pg_stat_database I/O times, or wait for it?
2. Does the patch need a release note or documentation hunk saying that blk_read_time and blk_write_time totals rise after the upgrade?
3. What does the pgbench comparison show with many clients and on another platform, where the clock source differs?
4. Can the vacuum and analyze snapshot of pgStatBlockReadTime and pgStatBlockWriteTime report a wrong elapsed time when track_io_timing changes mid-run?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner would be a committer who knows the cumulative statistics code and the I/O timing path in pgstat_io.c. The three visible messages show none taking part; the brief reports comments from Andres later in the thread.
- **Liability:** The owner signs for a one-time rise in reported blk_read_time and blk_write_time, a compile break for extensions that use the removed macros, and the per-I/O accounting path. The liability is small and may be short if the pgBufferUsage rework replaces these counters.
- **Author follow-up:** The authors posted v2 one day after the first review (message #3), and the CommitFest entry returned to a rebased state on 2026-10-09.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches five files, adds no user-visible surface, and closes a measured gap of about 15% in write time for users who run track_io_timing and compare pg_stat_database with pg_stat_io. The reported speed gain is about 0.5% in one single-client run, so the accuracy fix carries the case.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P4] covers per-backend I/O counters and was stopped over flush ordering between stats kinds, which this patch does not add. [P3] covers WAL in pg_stat_io. Message #13 points to the cumulative WAL statistics thread for a later pgBufferUsage rework, which is follow-up work and was not a prior rejection of this idea.
- standards-compat: 0 objection(s). No SQL syntax, GUC, catalog or pg_dump surface changes. The column types stay the same. The only user-visible change is higher, more accurate blk_read_time and blk_write_time totals after a major upgrade, and that corrects an undercount. Removing the macros and retyping pgStatBlockReadTime breaks extension builds loudly, which is normal for a major release. A release note for the step change in values is a level 4 item.
- replication-ondisk: 0 objection(s). The patch changes only backend-local accumulators and removes two macros. PgStat_StatDBEntry keeps PgStat_Counter microseconds, so the stats file, catalogs, WAL, replication and wire protocol do not change. The type change of pgStatBlockReadTime and pgStatBlockWriteTime in pgstat.h targets a major release only, and any extension that reads them fails to compile instead of misreading the values. Neither the on-disk stats layout nor that compile-time break could be confirmed without the code.
- security: 0 objection(s). The patch changes how existing pg_stat_database I/O timing counters accumulate inside a backend. It adds no view, column, function, privilege or GUC, and it does not change who can read these statistics. Higher-precision totals of a database's own I/O time do not give an unprivileged user a new timing or data channel, and per-backend memory use stays the same. I see no security or privilege angle.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The patch makes pg_stat_database accumulate I/O time in instr_time, as pg_stat_io, pgBufferUsage and function stats already do. Andres expects it to remove a conversion from the I/O hot path (message #5), and message #11 measures a small single-client gain. The later move to pgBufferUsage would replace these counters but gives the same precision, so users see no second change in values, and Andres asks for that move as separate work (message #9). Only the macro removal and the type change of pgStatBlockReadTime and pgStatBlockWriteTime touch the extension API, and a major release may change both. Neither [P3] nor [P4] fits: they dealt with view semantics changing between releases and with ordering between stats kinds at flush, and this patch introduces neither.
- already-exists: 0 objection(s). No existing feature gives per-database I/O times without the microsecond truncation: pg_stat_io is not per database, and pg_stat_statements is per query. The patch changes the type of an existing counter and adds no new mechanism; Andres (message #9) called the pgBufferUsage-based rework separate work, and Lukas (message #13) ties that rework to the open WAL statistics fix.
- demand: 0 objection(s). No demand objection applies. The change fixes a measured gap of about 15% in write time between pg_stat_database and pg_stat_io (message #1), costs non-users nothing, removes per-I/O work from a hot path that Andres expects to be a win (message #5), and adds no SQL, GUC or catalog surface. Whether to merge before or after the pgBufferUsage rework is a sequencing question for other angles.

### Precedents retrieved

- P1: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P2: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - used in the review; reason summarised from its thread
- P4: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - used in the review; reason summarised from its thread
- P5: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P6: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [pg_stat_database update stats_reset only by pg_stat_reset](https://commitfest.postgresql.org/patch/2116/) (cf, Withdrawn, 2019-07-19) - judged not relevant; reason summarised from its thread
- P8: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P9: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant
- P10: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P11: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P12: [Microsecond-based timeouts](https://commitfest.postgresql.org/patch/4245/) (cf, Returned with feedback, 2023-07-26) - judged not relevant
- P13: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P14: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant
- P15: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant
- P16: [pg_stat_lock: add blocker mode dimension](https://commitfest.postgresql.org/patch/6820/) (cf, Withdrawn, 2026-08-13) - judged not relevant
- P17: [In logs, use microsecond rather than millisecond granularity for timestamps](https://commitfest.postgresql.org/patch/3683/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P18: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P19: [Revert "Rename routines for write/read of pgstats file"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=381b3cfe2be02aa114ef7a28c4ec24feaf377c63) (revert, Reverted, 2026-07-14) - judged not relevant
- P20: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P21: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P22: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant
- P23: [Revert "Fix comments in instr_time.h and remove an unneeded cast to int64"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39a138fbef87803ab7fe494243c1ba36a093a0e8) (revert, Reverted, 2024-08-06) - judged not relevant
- P24: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 64 s; cost reported by the CLI: $1.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:44:04.803Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
