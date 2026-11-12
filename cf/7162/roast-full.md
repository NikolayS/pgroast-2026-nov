# Design pre-review: Add pg_stat_log_messages: cumulative statistics about server log messages (was: Add contrib module pg_stat_log: cumulative statistics about server log messages)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7162](https://commitfest.postgresql.org/patch/7162/) (Needs review); authors: Fabrízio Mello (fabriziomello)
- Thread: [Add contrib module pg_stat_log: cumulative statistics about server log messages](https://www.postgresql.org/message-id/flat/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), started 2026-08-18
- Messages read: 6 of 6
- Patch prose read: v1-0001-Add-pg_stat_log-contrib-module.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread and a concrete alerting use case, so level 1b does not stop the review. The design does: a fixed shared table that any role can fill (security-1, maintenance-fit-2) can drop the XX001 counts the proposal is justified by. The choice of a fixed-amount kind with a private hash table (maintenance-fit-1) and the log_min_messages floor (standards-compat-1) also need agreement before code review. Bertrand Drouvot's per-backend suggestion is still open.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 patch text was not available, so the exact column names, the lock used, the postmaster handling and the GUC context could not be checked. Only the first three of six messages were shown in full.

## The proposal, restated

**Problem.** Counting errors by SQLSTATE, database and user today requires parsing the server log, which is costly and lost after rotation. Monitoring roles often get pg_monitor but not log access, because logs can hold statements and sensitive data. Alerting on codes such as XX000, XX001 or XX002 has no counter source.

**Proposed design.** v2 adds a builtin fixed-amount stats kind, PGSTAT_KIND_LOGMSG, counted in EmitErrorReport() when a message reaches the server log. Entries are keyed by backend type, database, user, elevel and SQLSTATE in an index-based hash table inside the stats block. The table has a compile-time size of 64 entries per named errcode, and new combinations beyond capacity go to a dropped counter.

**User-visible surface.**
- New view pg_stat_log_messages (backend_type, database, user, elevel, sqlerrcode, sqlerrcode_name, count, stats_reset), readable with pg_read_all_stats.
- New enum GUC track_log_messages that sets the minimum severity to count; default none.
- pg_stat_reset_shared('log_messages') resets the counters.
- New function pg_stat_get_log_messages_dropped() reports combinations lost to a full table.
- Stats file grows by about 0.6 MB; counters persist across clean restart and are discarded after crash recovery.

**What the author claims.**
- Overhead in a log storm with many distinct SQLSTATEs was in the low single digits percent; the harness and numbers are not posted.
- Workloads that do not log see no measurable overhead.
- About 16,800 entries use about 0.6 MB of shared memory, allocated at startup even when tracking is off.
- The design complements Joe Conway's 2023 pg_stat_logmsg, which keyed messages by source location.
- v1 went through review and adversarial testing as an external extension.

**Thread state.** Kirk Wolak and Nikolay Samokhvalov support the idea, and Daniel Gustafsson's objection to contrib led to the v2 rewrite into core. Bertrand Drouvot suggests per-backend counters on the redesigned per-backend stats infrastructure; no one has reviewed v2 in detail, and it needs a rebase.

## 2. Prior attempts

### pg_stat_logmsg (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4431/](https://commitfest.postgresql.org/patch/4431/)
- **Thread:** [https://www.postgresql.org/message-id/flat/89742024-d51a-c66b-90b9-67f837072cd2@joeconway.com](https://www.postgresql.org/message-id/flat/89742024-d51a-c66b-90b9-67f837072cd2@joeconway.com)
- **Date:** decided 2024-10-07; started 2023-06-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tomas Vondra, Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch did not change after September 2023. In July 2024 reviewers asked what blocked it, questioned whether it belongs in contrib, and pointed out that line numbers change between minor releases, so old and new entries mix after an upgrade. Michael Paquier suggested building on the proposed pluggable cumulative statistics instead of copying the pg_stat_statements design, which does not scale well. The author agreed to put the version in the key and to rework the patch, but said time was the main blocker, and the visible messages show no new version before the entry was closed.
- **Reason (quote):** "The patch shares a lot of perks with pg_stat_statements that don't scale well." ([message](https://www.postgresql.org/message-id/Zpb9gBjB2FfiD8jO%40paquier.xyz))
- **What is different this time:** v2 keys by SQLSTATE, database, user and backend type, which stay stable across minor releases, unlike file name and line number. It is a builtin cumulative stats kind in core, not a contrib module that copies the pg_stat_statements design.

### SQL statements statistics counter view (pg_stat_sql) (related)

- **Link:** [https://commitfest.postgresql.org/patch/790/](https://commitfest.postgresql.org/patch/790/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJrrPGeY4xujjoR=z=KoyRMHEK_pSjjp=7VBhOAHq9rfgpV7QQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJrrPGeY4xujjoR=z=KoyRMHEK_pSjjp=7VBhOAHq9rfgpV7QQ@mail.gmail.com)
- **Date:** decided 2020-09-17; started 2016-08-20
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier returned the entry with feedback after months without activity, after Alvaro Herrera asked who would rebase it onto the new cmdtaglist.h enum tags. Andres Freund's 2017 review had left open questions: an extension might be the better place, tags should be enums in a fixed array, and counting belongs at the executor level. The author promised an array version, but the visible messages do not show one. A competing cmdstats patch by Mark Dilger also waited on benchmarks that Robert Haas requested.
- **Reason (quote):** "This patch had no activity for months, so I am marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/20200917050119.GI2873%40paquier.xyz))
- **What is different this time:** P1 counted SQL statements by command tag, not log messages by SQLSTATE. Both add a core counter view behind a track GUC.


## 3. Objections a reviewer is likely to raise

### 1. Unprivileged users can fill the shared table and hide later corruption errors

fixable · level 2, rubric item 7 · security / privileges · `security-1`

Any role runs a PL/pgSQL loop that raises errors with thousands of distinct made-up SQLSTATEs using raise ... using errcode. The fixed table of about 16,800 entries fills, and a later XX001 from another database lands only in the dropped counter. The operator finds out after the corruption, when the expected alert never fired.

Message #4 justifies core placement with alerting on XX000, XX001 and XX002. Message #5 sizes the table as one shared pool, keeps it full until a privileged reset, and lets new combinations fall into a single dropped counter. Raising an error with any SQLSTATE needs no privilege, so a non-privileged role controls what occupies the table. The dropped counter shows that something was lost, but not what.

Thread: [#4 (Nikolay Samokhvalov, 2026-09-09)](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5 (Fabrízio Mello, 2026-09-28)](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com)

**Question:** How does the design keep built-in error classes such as XX countable when a non-privileged role floods the table with user-defined SQLSTATEs?

Counter-check: held. Any role can raise errors with arbitrary SQLSTATEs, and ERROR reaches the server log by default. Message #5 describes one shared pool that stays full until a privileged reset, so the scenario works once tracking is on.

### 2. A full table can drop the corruption errors the feature exists for

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A multi-tenant cluster with thousands of roles records 28P01 and 42501 per role until all 16,800 slots are used. Weeks later an XX001 arrives, goes only into pg_stat_get_log_messages_dropped(), and the XX001 alert from message #4 never fires. The operator learns of this only after reading the logs.

Message #5 removed v1's max_entries GUC, so an operator cannot raise the capacity without rebuilding. The key includes user and database, so cardinality grows with roles and databases, while the size grows only with the number of SQLSTATE codes. The dropped counter shows that something was lost but not which SQLSTATE. Message #4 gives XX000, XX001 and XX002 alerting as the main use case.

Thread: [#4 (Nikolay Samokhvalov, 2026-09-09)](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5 (Fabrízio Mello, 2026-09-28)](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com)

**Question:** How does the design make sure rare high-severity SQLSTATEs are always counted on clusters with many roles or databases, without a rebuild?

Counter-check: held. The key includes user and database while capacity is fixed at compile time, and message #5 removed max_entries. The v1 example shows 28P01 with no user, so that part of the scenario is weak, but 42501 per role still fills the table.

### 3. A private hash table inside a fixed-amount stats kind

fixable · level 2, rubric item 12 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later change to pgstat snapshot or persistence code assumes fixed-amount kinds are small flat structs. It breaks the index-chained table, and no test outside stats.sql fails. Meanwhile every cluster allocates about 0.6 MiB of shared memory and writes it to the stats file at shutdown, even with track_log_messages = none.

Existing fixed-amount kinds such as archiver, bgwriter, io and wal are small fixed counters. Keyed data normally uses variable-numbered kinds in dshash. Message #5 keeps a bespoke hash table only because the fixed-amount memcpy snapshot rules out dshash, so pgstat would carry a second hashing scheme that only this feature uses. The table is also sized from the errcodes.txt count at compile time, so adding SQLSTATEs changes shared memory and the stats file size without anyone choosing to.

Thread: [#1 (Fabrizio Mello, 2026-08-18)](https://www.postgresql.org/message-id/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), [#5 (Fabrízio Mello, 2026-09-28)](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com)

**Question:** Why can a variable-numbered kind, or a hybrid with a small fixed index, not hold these keys, and who keeps the custom table in sync with pgstat changes?

Counter-check: held. The author explains why dshash does not fit a fixed-amount kind, but not why a variable-numbered kind was not used. The 0.6 MiB allocated with tracking off is confirmed in message #5; the errcodes.txt part is covered by replication-ondisk-2.

### 4. Counters silently depend on log_min_messages, a separate logging setting

fixable · level 2, rubric item 10 · new GUC · `standards-compat-1`

A managed-platform operator raises log_min_messages to fatal, or sets it per role with alter role ... set, to cut log volume. track_log_messages = error stays on, but ERROR-level XX001 and XX000 are no longer counted for those sessions. The corruption alert from message #4 never fires, and nobody sees why until they read the docs.

Message #5 says log_min_messages acts as a floor on what is counted, so the meaning of track_log_messages depends on a second GUC that can differ per role and per database. Other track_* settings do not change meaning based on logging configuration. This coupling works against the use case in message #4, where monitoring should not depend on log access or log setup. Messages sent only to the client are never counted.

Thread: [#4 (Nikolay Samokhvalov, 2026-09-09)](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5 (Fabrízio Mello, 2026-09-28)](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com)

**Question:** Why should a statistics counter depend on log_min_messages instead of counting at error-report time, regardless of where the message goes?

Counter-check: held. Messages #1 and #5 state the log_min_messages floor but give no reason for it. A superuser can set log_min_messages per role or database, so the scenario can happen and conflicts with the alerting use case in message #4.

### 5. One shared lock in EmitErrorReport() for every process, no numbers posted

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-3`

A connection storm with a bad password makes hundreds of backends emit FATAL 28P01 at once. Each one takes the same LWLock to count, so authentication failures serialize and the error path slows down when the system is under the most stress. The postmaster also logs through this path and would wait on a shared-memory lock.

Message #1 states the counting takes one LWLock, and v2 moves this into EmitErrorReport() for all process types. The only measurement is a claim of low single-digit overhead in message #1, with no harness or numbers for either version. The postmaster normally avoids shared-memory locks so that it survives damaged shared state, and the proposal does not say how v2 handles that.

Thread: [#1 (Fabrizio Mello, 2026-08-18)](https://www.postgresql.org/message-id/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), [#5 (Fabrízio Mello, 2026-09-28)](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com)

**Question:** What overhead do you measure with many concurrent backends logging, and does the postmaster or a backend in a critical section ever take the lock?

Counter-check: held. Message #1 confirms one LWLock per counted message and says the module handles the postmaster, without saying how. No harness or numbers are posted, so the concurrency and postmaster questions are open, though only for clusters that turn tracking on.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The stats file gains a block of about 0.6 MiB whose size follows the errcodes.txt count. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds the system view pg_stat_log_messages and the function pg_stat_get_log_messages_dropped(). |
| ABI / extension API | **yes** | The patch adds a builtin stats kind, PGSTAT_KIND_LOGMSG, with its own hash table inside pgstat. |
| SQL-standard conflict or non-standard syntax | **yes** | The brief lists a view column named user, which is a reserved SQL keyword. |
| new GUC | **yes** | The patch adds the enum GUC track_log_messages, default none. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Any role can raise errors that take slots in the shared table; reads need pg_read_all_stats. |
| backpatchability | **yes** | An errcode added in a back branch would change the shared memory and stats file layout. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Core would carry a new view, GUC, stats kind and a hash table used by this feature only. |
| performance on the hot path | **yes** | Counting takes an LWLock in EmitErrorReport() for every tracked message in every process type. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A working out-of-tree extension gives the same counters through emit_log_hook. |

## 5. Questions the author must answer before anyone reviews code

1. How does the design keep XX000, XX001 and XX002 countable once the table is full, whether from many roles or from one role raising made-up SQLSTATEs?
2. Why can a variable-numbered stats kind, or the per-backend stats that Bertrand Drouvot suggested, not hold these keys?
3. Why does counting stop at the log_min_messages floor instead of counting at error-report time, wherever the message goes?
4. What overhead do you measure with many backends logging at once, and does the postmaster or a backend in a critical section take the lock?
5. What makes a stats file written with a different errcode count fail loudly on read, without a manual PGSTAT_FILE_FORMAT_ID bump?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner would need to know the cumulative statistics code and the error-reporting path in elog.c. Daniel Gustafsson commented on placement only, and nobody has reviewed v2 in detail.
- **Liability:** A permanent core view and GUC, a stats file layout tied to errcodes.txt, and a lock taken in EmitErrorReport() by every process type, for as long as the view exists.
- **Author follow-up:** The author rewrote the patch into core after message #3 and maintains the out-of-tree extension. The entry has needed a rebase since 2026-10-04.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Operators whose monitoring roles have no log access benefit, and the thread shows support. The capacity and stats-kind questions decide the shape of the patch, so detailed review should wait for them; review cost is an estimate because the v2 patch was not read.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) View column names depart from datid/datname/usesysid/usename convention. Not listed: not among the strongest objections.
- `replication-ondisk-2` (fixable, level 2) Stats block size tied to errcodes.txt without a format version check. Not listed: not among the strongest objections.
- `already-exists-1` (fixable, level 1b) Working extension already exists; core adds distribution, not capability. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-3` Table size derived from errcodes.txt ties stats layout to SQLSTATE list. Refuted (not specific): Duplicate of replication-ondisk-2, which states the same errcodes.txt layout coupling with a more exact failure path and also covers the back-branch case.
- `replication-ondisk-1` Crash recovery discards the counters the motivating alerts depend on. Refuted (no failure scenario): Every cumulative statistics kind discards its counters after crash recovery, so the patch follows existing practice. Monitoring can detect a crash restart by other means, and what remains is a documentation note about PANIC rows, which is out of scope.

### Review angles

- prior-attempts: 0 objection(s). The closest precedent, pg_stat_logmsg [P3], which message #1 cites as its inspiration, was set aside over source-location keys that change between minor releases, a pg_stat_statements-style design and the contrib question. v2 answers each point: it uses stable SQLSTATE keys and the cumulative stats API, and moved into core after message #3. The missing benchmarks echo [P1], but that point belongs to the performance angle and rests only on an AI summary.
- standards-compat: 3 objection(s)
- replication-ondisk: 2 objection(s). The proposal adds no WAL records, catalog changes or wire protocol changes. Its only persistent state is the cumulative stats file, so the objections above cover that file alone.
- security: 1 objection(s). Read access through pg_read_all_stats and reset through pg_stat_reset_shared follow existing practice. The material does not say whether track_log_messages is superuser-only, so that point is left as a question.
- maintenance-fit: 3 objection(s). Objections listed; the idea itself has demand in messages #2 and #4.
- already-exists: 1 objection(s). One objection only. pg_stat_database counts deadlocks, rollbacks, conflicts and checksum failures but has no general SQLSTATE counter, so this objection rests on the existing extension, not on core views.
- demand: 0 objection(s). The material shows real demand that has lasted. The 2023 pg_stat_logmsg proposal [P3], this author and three thread participants want error counters, and message #4 gives a concrete alerting case (XX000/XX001/XX002) for managed setups where monitoring roles get pg_monitor but not log access. Cost falls mostly on users because counting is off by default, apart from the 0.6 MB of shared memory and stats file space that every instance pays.

### Precedents retrieved

- P1: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - used in the review; reason summarised from its thread
- P2: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant; reason summarised from its thread
- P3: [pg_stat_logmsg](https://commitfest.postgresql.org/patch/4431/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P4: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P5: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - judged not relevant; reason summarised from its thread
- P6: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P7: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant; reason summarised from its thread
- P8: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Issue a log message when the backtrace logged is cut off](https://commitfest.postgresql.org/patch/3547/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P11: [Reduce the log spam by excluding certain SQLSTATEs](https://commitfest.postgresql.org/patch/4964/) (cf, Withdrawn, 2025-04-03) - judged not relevant
- P12: [SQL function to report log message](https://commitfest.postgresql.org/patch/314/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P13: [Revert "Add GUC backtrace_on_internal_error"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=592a2283721f7143999364ef487f2b4993f5161d) (revert, Reverted, 2024-04-29) - judged not relevant
- P14: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant
- P15: [Revert error-throwing wrappers for the printf family of functions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c071936e94c6859afb2ec8d2c8dddf7bcdab7ee) (revert, Reverted, 2015-05-19) - judged not relevant
- P16: [Checkpoint extension hook](https://commitfest.postgresql.org/patch/4868/) (cf, Returned with feedback, 2024-07-23) - judged not relevant
- P17: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P18: [Extensible storage manager API - smgr hooks](https://commitfest.postgresql.org/patch/3216/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P19: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant
- P21: [Add a statistics view to track usage of deprecated features](https://commitfest.postgresql.org/patch/6948/) (cf, Withdrawn, 2026-07-03) - used in the review
- P22: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P23: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P24: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 178 s; cost reported by the CLI: $2.34
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 105 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:37.555Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
