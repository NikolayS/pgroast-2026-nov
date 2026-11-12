# Design pre-review: Add pg_stat_log_messages: cumulative statistics about server log messages (was: Add contrib module pg_stat_log: cumulative statistics about server log messages)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7162](https://commitfest.postgresql.org/patch/7162/); [thread](https://www.postgresql.org/message-id/flat/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread and a concrete alerting use case, so level 1b does not stop the review. The design does: a fixed shared table that any role can fill (security-1, maintenance-fit-2) can drop the XX001 counts the proposal is justified by. The choice of a fixed-amount kind with a private hash table (maintenance-fit-1) and the log_min_messages floor (standards-compat-1) also need agreement before code review. Bertrand Drouvot's per-backend suggestion is still open.

Not checked: The v2 patch text was not available, so the exact column names, the lock used, the postmaster handling and the GUC context could not be checked. Only the first three of six messages were shown in full.

## Prior attempts

- [pg_stat_logmsg](https://www.postgresql.org/message-id/flat/89742024-d51a-c66b-90b9-67f837072cd2@joeconway.com) (similar idea; Returned with feedback, 2024-10-07). "The patch shares a lot of perks with pg_stat_statements that don't scale well." ([message](https://www.postgresql.org/message-id/Zpb9gBjB2FfiD8jO%40paquier.xyz)) Different now: v2 keys by SQLSTATE, database, user and backend type, which stay stable across minor releases, unlike file name and line number. It is a builtin cumulative stats kind in core, not a contrib module that copies the pg_stat_statements design.
- [SQL statements statistics counter view (pg_stat_sql)](https://www.postgresql.org/message-id/flat/CAJrrPGeY4xujjoR=z=KoyRMHEK_pSjjp=7VBhOAHq9rfgpV7QQ@mail.gmail.com) (related; Returned with feedback, 2020-09-17). "This patch had no activity for months, so I am marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/20200917050119.GI2873%40paquier.xyz)) Different now: P1 counted SQL statements by command tag, not log messages by SQLSTATE. Both add a core counter view behind a track GUC.

## Objections a reviewer is likely to raise

### 1. Unprivileged users can fill the shared table and hide later corruption errors

Any role runs a PL/pgSQL loop that raises errors with thousands of distinct made-up SQLSTATEs using raise ... using errcode. The fixed table of about 16,800 entries fills, and a later XX001 from another database lands only in the dropped counter. The operator finds out after the corruption, when the expected alert never fired.

Message #4 justifies core placement with alerting on XX000, XX001 and XX002. Message #5 sizes the table as one shared pool, keeps it full until a privileged reset, and lets new combinations fall into a single dropped counter. Raising an error with any SQLSTATE needs no privilege, so a non-privileged role controls what occupies the table. The dropped counter shows that something was lost, but not what.

**Question:** How does the design keep built-in error classes such as XX countable when a non-privileged role floods the table with user-defined SQLSTATEs?

<sub>fixable, level 2, security / privileges; messages [#4](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com); `security-1`</sub>

### 2. A full table can drop the corruption errors the feature exists for

A multi-tenant cluster with thousands of roles records 28P01 and 42501 per role until all 16,800 slots are used. Weeks later an XX001 arrives, goes only into pg_stat_get_log_messages_dropped(), and the XX001 alert from message #4 never fires. The operator learns of this only after reading the logs.

Message #5 removed v1's max_entries GUC, so an operator cannot raise the capacity without rebuilding. The key includes user and database, so cardinality grows with roles and databases, while the size grows only with the number of SQLSTATE codes. The dropped counter shows that something was lost but not which SQLSTATE. Message #4 gives XX000, XX001 and XX002 alerting as the main use case.

**Question:** How does the design make sure rare high-severity SQLSTATEs are always counted on clusters with many roles or databases, without a rebuild?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. A private hash table inside a fixed-amount stats kind

A later change to pgstat snapshot or persistence code assumes fixed-amount kinds are small flat structs. It breaks the index-chained table, and no test outside stats.sql fails. Meanwhile every cluster allocates about 0.6 MiB of shared memory and writes it to the stats file at shutdown, even with track_log_messages = none.

Existing fixed-amount kinds such as archiver, bgwriter, io and wal are small fixed counters. Keyed data normally uses variable-numbered kinds in dshash. Message #5 keeps a bespoke hash table only because the fixed-amount memcpy snapshot rules out dshash, so pgstat would carry a second hashing scheme that only this feature uses. The table is also sized from the errcodes.txt count at compile time, so adding SQLSTATEs changes shared memory and the stats file size without anyone choosing to.

**Question:** Why can a variable-numbered kind, or a hybrid with a small fixed index, not hold these keys, and who keeps the custom table in sync with pgstat changes?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. Counters silently depend on log_min_messages, a separate logging setting

A managed-platform operator raises log_min_messages to fatal, or sets it per role with alter role ... set, to cut log volume. track_log_messages = error stays on, but ERROR-level XX001 and XX000 are no longer counted for those sessions. The corruption alert from message #4 never fires, and nobody sees why until they read the docs.

Message #5 says log_min_messages acts as a floor on what is counted, so the meaning of track_log_messages depends on a second GUC that can differ per role and per database. Other track_* settings do not change meaning based on logging configuration. This coupling works against the use case in message #4, where monitoring should not depend on log access or log setup. Messages sent only to the client are never counted.

**Question:** Why should a statistics counter depend on log_min_messages instead of counting at error-report time, regardless of where the message goes?

<sub>fixable, level 2, new GUC; messages [#4](https://www.postgresql.org/message-id/CAM527d9FX3FL7P63Ww9qVQfaqWoUDMMAp_GgSxWg%2BO2HWdigeQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com); `standards-compat-1`</sub>

### 5. One shared lock in EmitErrorReport() for every process, no numbers posted

A connection storm with a bad password makes hundreds of backends emit FATAL 28P01 at once. Each one takes the same LWLock to count, so authentication failures serialize and the error path slows down when the system is under the most stress. The postmaster also logs through this path and would wait on a shared-memory lock.

Message #1 states the counting takes one LWLock, and v2 moves this into EmitErrorReport() for all process types. The only measurement is a claim of low single-digit overhead in message #1, with no harness or numbers for either version. The postmaster normally avoids shared-memory locks so that it survives damaged shared state, and the proposal does not say how v2 handles that.

**Question:** What overhead do you measure with many concurrent backends logging, and does the postmaster or a backend in a critical section ever take the lock?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CABo-N96sjT0KwtCFVBkuMQEAfe-834Sd2zjbv9qKJORXkf%3DBiA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABo-N94G1PgQs%3DicPK%2BOvCchKqvVLsHLCv-sWDO2veYeRpo%2BvQ%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. How does the design keep XX000, XX001 and XX002 countable once the table is full, whether from many roles or from one role raising made-up SQLSTATEs?
2. Why can a variable-numbered stats kind, or the per-backend stats that Bertrand Drouvot suggested, not hold these keys?
3. Why does counting stop at the log_min_messages floor instead of counting at error-report time, wherever the message goes?
4. What overhead do you measure with many backends logging at once, and does the postmaster or a backend in a critical section take the lock?
5. What makes a stats file written with a different errcode count fail loudly on read, without a manual PGSTAT_FILE_FORMAT_ID bump?

## Touch points

- **on-disk format / pg_upgrade:** The stats file gains a block of about 0.6 MiB whose size follows the errcodes.txt count.
- **catalog changes:** The patch adds the system view pg_stat_log_messages and the function pg_stat_get_log_messages_dropped().
- **ABI / extension API:** The patch adds a builtin stats kind, PGSTAT_KIND_LOGMSG, with its own hash table inside pgstat.
- **SQL-standard conflict or non-standard syntax:** The brief lists a view column named user, which is a reserved SQL keyword.
- **new GUC:** The patch adds the enum GUC track_log_messages, default none.
- **security / privileges:** Any role can raise errors that take slots in the shared table; reads need pg_read_all_stats.
- **backpatchability:** An errcode added in a back branch would change the shared memory and stats file layout.
- **maintenance burden ("who maintains this in 10 years"):** Core would carry a new view, GUC, stats kind and a hash table used by this feature only.
- **performance on the hot path:** Counting takes an LWLock in EmitErrorReport() for every tracked message in every process type.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A working out-of-tree extension gives the same counters through emit_log_hook.

## Who would commit it, and when

- Owner: silent. The owner would need to know the cumulative statistics code and the error-reporting path in elog.c. Daniel Gustafsson commented on placement only, and nobody has reviewed v2 in detail.
- Priority: fix design first; audience a sizeable minority; review cost medium. Operators whose monitoring roles have no log access benefit, and the thread shows support. The capacity and stats-kind questions decide the shape of the patch, so detailed review should wait for them; review cost is an estimate because the v2 patch was not read.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
