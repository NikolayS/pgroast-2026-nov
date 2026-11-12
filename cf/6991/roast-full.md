# Design pre-review: Add per-backend AIO statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6991](https://commitfest.postgresql.org/patch/6991/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Add per-backend AIO statistics](https://www.postgresql.org/message-id/flat/akzcq9jtWK5C92g1%40bdtpg), started 2026-07-07
- Messages read: 8 of 8
- Patch prose read: v1-0001-Add-per-backend-AIO-statistics.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: high)

No reviewer disputes the goal, but the thread has already set aside the v1 design. In messages #5 to #7 the author agreed to drop three derivable counters and to build the exited-plus-live stats infrastructure first, yet the CF entry still holds v1 (maintenance-fit-1, standards-compat-2). v1 also loses the counters of exited backends (maintenance-fit-2). Code review makes sense once a version exists on the agreed design.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact cost of the counter calls and the permission checks of pg_stat_get_backend_aio() are unchecked. Messages #4 to #8 were seen only through the brief and the objections.

## The proposal, restated

**Problem.** Users cannot see how AIO ran over time. pg_aios shows only in-flight handles, and pg_stat_get_backend_io() counts IO volume, but not whether IOs ran sync or async, whether a backend waited for free handles, or which backends processed completions.

**Proposed design.** The patch adds cumulative AIO counters to the existing backend statistics kind (pgstat_backend.c), incremented from the AIO staging, completion and submit paths. pgstat_flush_backend() gets a new flag to flush the AIO counters. There is no global view, so stats of exited backends are lost.

**User-visible surface.**
- New function pg_stat_get_backend_aio(integer) returning one record per PID.
- Columns: started, executed_sync, executed_async, completed_self, completed_other, handle_waits, submitted, stats_reset.
- No data for checkpointer, background writer, startup process and autovacuum launcher.
- completed_other is documented as meaningful only with io_method = io_uring.
- New flag value for pgstat_flush_backend() in the backend stats C API.

**What the author claims.**
- The counters help tune AIO, for example handle_waits suggesting io_max_concurrency is too low.
- The author observed no measurable regression; the cost is about 4 function calls and counter increments per IO.
- Per-backend memory cost is 56 bytes for PgStat_AioCounters.
- executed_sync and executed_async match the normal and bulkread read counts of pg_stat_get_backend_io() in his example.
- No PGSTAT_FILE_FORMAT_ID bump is needed because backend stats are never written to disk.

**Thread state.** Andres Freund questioned derivable counters, the lack of stats for exited backends, and pgstat_backend.c as a direction, proposing shared exited-backend plus live per-backend counters. The author agreed to do that redesign first in a new thread; the CF entry still holds v1, which a reviewer tested functionally.

## 2. Prior attempts

### Revert "Track statistics for spilling of changes from ReorderBuffer". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+fd4k5_pPAYRTDrO2PbtTOe0eHQpBvuqmCr8ic39uTNmR49Eg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+fd4k5_pPAYRTDrO2PbtTOe0eHQpBvuqmCr8ic39uTNmR49Eg@mail.gmail.com), [https://www.postgresql.org/message-id/flat/20200602.161518.1399689010416646074.horikyota.ntt@gmail.com](https://www.postgresql.org/message-id/flat/20200602.161518.1399689010416646074.horikyota.ntt@gmail.com), [https://www.postgresql.org/message-id/flat/CA+fd4k4vNg7dRO5ECHdtQXXf1=Q4M98pfLW0dU7BKD8h79pkqA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+fd4k4vNg7dRO5ECHdtQXXf1=Q4M98pfLW0dU7BKD8h79pkqA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/688b0b7f-2f6c-d827-c27b-216a8e3ea700@2ndquadrant.com](https://www.postgresql.org/message-id/flat/688b0b7f-2f6c-d827-c27b-216a8e3ea700@2ndquadrant.com)
- **Date:** decided 2020-07-13; started 2019-11-21
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** The stats with this commit was available only for WALSenders, however, users might want to see for backends doing logical decoding via SQL API. Then, users might want to reset and access these stats across server restart which was not possible with the current patch. List of commits reverted: caa3c4242c Don't call elog() while holding spinlock. e641b2a995 Doc: Update the documentation for spilled transaction statistics. 5883f5fe27 Fix unportable printf format introduced in commit 9290ad198. 9290ad198b Track statistics for spilling of changes from ReorderBuffer. Additionaly, remove the release notes entry for this feature.
- **Reason (quote):** "The stats with this commit was available only for WALSenders, however, users might want to see for backends doing logical decoding via SQL API."
- **What is different this time:** P5 tracked logical decoding spill stats; this tracks AIO counters. Both cover only some processes and lose data: v1 drops counters at backend exit and skips IO workers and several auxiliary processes.

### Show WAL stats on pg_stat_io (related)

- **Link:** [https://commitfest.postgresql.org/patch/4416/](https://commitfest.postgresql.org/patch/4416/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com)
- **Date:** decided 2024-01-18; started 2023-06-28
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** pg_stat_io ties op_bytes to a fixed block size per object and context, but some WAL reads, such as those by the WAL receiver, have a variable size. Reviewers did not agree whether to set op_bytes to 1, to NULL, or to add per-operation byte columns. Michael Paquier proposed to set the patch aside until WAL reads and writes, async I/O and cached reads are settled, so that the meaning of pg_stat_io does not change between releases. The author agreed, and the patch was marked Returned with feedback.
- **Reason (quote):** "IMHO, perhaps we'd better put this patch aside until we are absolutely *sure* of what we want to achieve when it comes to WAL, and I am afraid that this cannot happen until we're happy with the way we handle WAL reads *and* writes, including WAL receiver or anything that has the idea of pulling its own page callback with XLogReaderAllocate() in the backend." ([message](https://www.postgresql.org/message-id/ZaTQSOVpwekhOXx-%40paquier.xyz))
- **What is different this time:** P6 added WAL I/O to pg_stat_io and was set aside until the wider I/O stats design settled. Here the author agreed in message #7 to do the exited-plus-live stats redesign before the AIO counters.


## 3. Objections a reviewer is likely to raise

### 1. v1 grows pgstat_backend.c, a direction the thread already questioned

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A committer applies v1 from the CommitFest entry, and the AIO counters become one more stats family inside the backend stats kind. Each later per-backend counter set lands in the same file with its own flush flag. The planned exited-plus-live redesign then has to move or rewrite all of them.

In message #5, Andres Freund doubts that pgstat_backend.c is the right direction because it will collect "all kinds of stats". He proposes per-kind shared counters for exited backends plus live per-backend counters. In message #7 the author agreed to do that redesign first in a new thread. The CF entry still holds v1, which uses the design both sides set aside.

Thread: [#5 (Andres Freund, 2026-07-08)](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6 (Bertrand Drouvot, 2026-07-09)](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg), [#7 (Bertrand Drouvot, 2026-07-10)](https://www.postgresql.org/message-id/alB7acgJ%2FWW7B1yC%40bdtpg)

**Question:** Should this CF entry be withdrawn until the exited-plus-live infrastructure exists, so that the AIO counters are built on it rather than on pgstat_backend.c?

Counter-check: held. Messages #5 and #7 confirm that both sides set aside the pgstat_backend.c design, and the CF entry still holds v1. A version built on the redesign would answer this, so the severity is fixable.

### 2. v1 column set conflicts with the agreed redesign

fixable · level 2, rubric item 6 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

v1 is committed from the open CF entry. Monitoring tools then query started, executed_async and completed_self. The redesign in message #7 drops these derivable columns and adds a total across exited backends, so those queries break or change meaning in the next release.

The author agreed in message #6 to remove started, executed_async and completed_self, and agreed in message #7 to land the redesign first. The CF entry still offers v1 as "Needs review". In [P6] a pg_stat_io extension was set aside so that the view's meaning would not change between releases, and the same reason applies to this function's contract. Exposing v1 now would set a column list that the thread has already rejected.

**Precedents.**
- [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (Returned with feedback, 2024-01-18); thread: [link](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com)

Thread: [#5 (Andres Freund, 2026-07-08)](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6 (Bertrand Drouvot, 2026-07-09)](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg), [#7 (Bertrand Drouvot, 2026-07-10)](https://www.postgresql.org/message-id/alB7acgJ%2FWW7B1yC%40bdtpg)

**Question:** Should this CF entry be withdrawn until a version exists with the reduced column set and an aggregate over exited backends?

Counter-check: held. The author agreed in message #6 to drop the derivable columns, but no revised patch exists and the CF entry still holds v1. P6 is only a loose parallel, and the objection stands without it.

### 3. Completion columns change meaning with io_method, silently zero by default

fixable · level 2, rubric item 10 · design, new GUC · `standards-compat-1`

A DBA on a server with io_method = worker, the default, reads completed_other = 0 and a small completed_self. The DBA concludes that completions are not shared or that IOs did not complete. The numbers never say that worker mode hides completions, so the DBA learns this only from a note in the docs.

The doc hunk calls completed_other meaningful only under io_uring, because IO workers do not track these statistics. Message #8 likely ran in worker mode, and it shows completed_self = 224 against executed_async = 26495. So completed_self also undercounts there, and the docs do not say so. Under the default setting the columns return plausible numbers with no sign that they are incomplete, and the meaning of the columns depends on a GUC.

Thread: [#1 (Bertrand Drouvot, 2026-07-07)](https://www.postgresql.org/message-id/akzcq9jtWK5C92g1%40bdtpg), [#8 (solai v, 2026-07-30)](https://www.postgresql.org/message-id/CAF0whufYfeLqsUy%2BUOoiYZgLpsPEYfXJm%3DqNocL-k69-yQbczw%40mail.gmail.com)

**Question:** Under io_method = worker, what should completed_self and completed_other report, and why not return NULL or count completions on the worker side?

Counter-check: held. The doc hunk limits only completed_other to io_uring. Message #8 shows completed_self = 224 against executed_async = 26495, and the docs do not explain that gap.

### 4. Per-IO counter calls on AIO hot paths have no published measurement

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-3`

Every server pays four function calls and counter updates per IO in pgaio_io_stage(), pgaio_io_process_completion() and pgaio_submit_staged(), whether or not anyone reads the function. On a high-IOPS io_uring workload, a small regression would appear only after release, with no numbers to compare against.

Message #3 asks for a review of these call sites because they run for all workloads by default. Message #1 reports no measurable regression but gives no workload or numbers. Andres Freund in message #5 says the overhead "might be ok" but questions the number of counters per IO. If ten monitoring patches each add a few calls per IO, the AIO path slows for everyone.

Thread: [#1 (Bertrand Drouvot, 2026-07-07)](https://www.postgresql.org/message-id/akzcq9jtWK5C92g1%40bdtpg), [#3 (Michael Paquier, 2026-07-08)](https://www.postgresql.org/message-id/ak3zpA2HHh5CxQK8%40paquier.xyz), [#5 (Andres Freund, 2026-07-08)](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi)

**Question:** What does a high-IOPS benchmark of small random reads under io_uring and worker mode show, with and without the reduced counter set?

Counter-check: held. Message #3 asks about the cost at these call sites, and message #1 gives no workload or numbers. Andres Freund says the overhead "might be ok" and the pattern copies pgstat_count_backend_io_op(), so this is a request for numbers.

### 5. Per-backend-only function fixes a user surface the redesign will change

fixable · level 1b, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-2`

A DBA monitors pg_stat_get_backend_aio() across parallel queries. The parallel workers exit, so their counters disappear, and the totals undercount AIO activity. If a global view arrives in a later release, the existing function and its columns must stay, or users' monitoring queries break.

The proposal has no global view, loses the stats of exited backends, and reports nothing for the checkpointer, bgwriter, startup process and autovacuum launcher. Andres Freund says in message #5 that stats for exited backends are almost always wanted. [P5] was reverted partly because its stats covered only some process types and not all users who needed them. Shipping the narrow function first commits the project to supporting it next to whatever the redesign produces.

**Precedents.**
- [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (Reverted, 2020-07-13); thread: [#1](https://www.postgresql.org/message-id/flat/CA+fd4k5_pPAYRTDrO2PbtTOe0eHQpBvuqmCr8ic39uTNmR49Eg@mail.gmail.com), [#2](https://www.postgresql.org/message-id/flat/20200602.161518.1399689010416646074.horikyota.ntt@gmail.com), [#3](https://www.postgresql.org/message-id/flat/CA+fd4k4vNg7dRO5ECHdtQXXf1=Q4M98pfLW0dU7BKD8h79pkqA@mail.gmail.com), [#4](https://www.postgresql.org/message-id/flat/688b0b7f-2f6c-d827-c27b-216a8e3ea700@2ndquadrant.com)

Thread: [#5 (Andres Freund, 2026-07-08)](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6 (Bertrand Drouvot, 2026-07-09)](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg)

**Question:** Can the function and its column set be defined so that they stay unchanged once exited-backend totals and a global view exist?

Counter-check: held. Message #5 says stats for exited backends, such as parallel workers, are almost always wanted, and v1 loses them. The author's global view from message #6 is in no patch, and P5 records a comparable coverage gap.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a pg_proc.dat entry for pg_stat_get_backend_aio() and needs a catversion bump. |
| ABI / extension API | **yes** | pgstat_flush_backend() gets a new flag value, and pgstat.h gains the PgStat_AioCounters struct. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: v1 column set conflicts with the agreed redesign |
| new GUC | **yes** | see objection standards-compat-1: Completion columns change meaning with io_method, silently zero by default |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new stats family in pgstat_backend.c and a seven-column function contract must be kept through the planned stats redesign. |
| performance on the hot path | **yes** | Counter calls run per IO in pgaio_io_stage(), pgaio_io_process_completion() and pgaio_submit_staged() for every workload. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should this CF entry be withdrawn or marked Waiting on Author until the exited-plus-live stats infrastructure exists and a version is built on it?
2. Which columns remain after dropping started, executed_async and completed_self, and will that list stay unchanged once exited-backend totals and a global view exist?
3. Under io_method = worker, what should completed_self and completed_other report, and why not return NULL or count completions in the IO workers?
4. What does a high-IOPS benchmark of small random reads show under io_uring and worker mode, with and without the reduced counter set?
5. How will users see AIO counters for parallel workers and other backends that have exited before the function is called?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund (AIO) and Michael Paquier (cumulative stats) take part in the thread; the owner needs to know both the AIO paths and the pgstat infrastructure.
- **Liability:** The owner signs for per-IO counter calls in the AIO staging, completion and submit paths on every server, and for a function column list that monitoring tools will query across releases.
- **Author follow-up:** The author agreed in messages #6 and #7 to revise the counters and do the redesign in a new thread, and moved the CF entry forward in October.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The counters serve people who tune AIO, and the 19.5 KB patch touches aio.c hot paths and the backend stats kind. The thread agreed to redesign the stats infrastructure first, so reviewing v1 now would spend time on a design both sides set aside.

## Appendix

### Refuted in the counter-check

- `demand-1` No user demand shown beyond the author's own experiments. Refuted (not specific): The per-IO cost part is a duplicate of maintenance-fit-3, and the derivable-counter part is a duplicate of standards-compat-2. The rest does not hold: pg_aios and pg_stat_get_backend_io() cannot report handle waits or sync fallback, and no reviewer disputed the goal.

### Review angles

- prior-attempts: 0 objection(s). The prior-attempt concern is coverage that P5's revert recorded: stats for exited backends and for IO workers. Andres raised it in message #5, and in message #7 the author chose to do the exited-plus-live redesign first, so the thread has settled it. The CF entry still holds v1 under Needs review, but the thread has replaced it. Without a new version, there is nothing new to object to from this angle.
- standards-compat: 2 objection(s). The function name, the per-PID function form and the stats_reset column follow pg_stat_get_backend_io(). No new GUC, syntax or dumped object is added.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state. Backend stats are never written to disk, so there is no stats file format change and nothing for pg_upgrade to handle. The catalog change is one new pg_proc entry with a catversion bump, which is routine for a major release. There is no WAL, replication or wire protocol change. The pgstat_flush_backend() flag and struct changes are internal and land only in a new major version, so ABI and backpatching are not affected.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. pg_stat_get_backend_aio() exposes only per-PID IO counts, follows the same access pattern as the existing pg_stat_get_backend_io(), and adds no new privilege, escalation path, RLS bypass, or unbounded resource use an unprivileged user could drive. Not checked without the code: whether its permission checks match pg_stat_get_backend_io() exactly.
- maintenance-fit: 3 objection(s). Objections found; see list.
- already-exists: 0 objection(s). No existing feature, view or extension gives cumulative sync versus async execution, handle waits or cross-backend completion counts; pg_aios shows only in-flight handles and pg_stat_get_backend_io() counts volume by context. Two overlap points were raised and accepted in the thread: the derivable counters and the duplicate per-backend mechanism in pgstat_backend.c (messages #5 to #7).
- demand: 1 objection(s). One demand objection only. The redesign that the author agreed to in message #7 may change which counters exist, so demand should be judged again against that version.

### Precedents retrieved

- P1: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - used in the review; reason summarised from its thread
- P2: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P3: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant; reason summarised from its thread
- P4: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P5: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - used in the review
- P6: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - used in the review; reason summarised from its thread
- P7: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P8: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P9: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P10: [REINDEX backend filtering](https://commitfest.postgresql.org/patch/2895/) (cf, Withdrawn, 2021-05-08) - judged not relevant; reason summarised from its thread
- P11: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P12: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - judged not relevant
- P13: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P14: [Add a statistics view to track usage of deprecated features](https://commitfest.postgresql.org/patch/6948/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P15: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P16: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P17: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P18: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant
- P19: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P20: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P21: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P22: [pg_stat_walwrites system view](https://commitfest.postgresql.org/patch/1003/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P23: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant
- P24: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 147 s; cost reported by the CLI: $1.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:13:12.174Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
