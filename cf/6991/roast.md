# Design pre-review: Add per-backend AIO statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6991](https://commitfest.postgresql.org/patch/6991/); [thread](https://www.postgresql.org/message-id/flat/akzcq9jtWK5C92g1%40bdtpg), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: high). No reviewer disputes the goal, but the thread has already set aside the v1 design. In messages #5 to #7 the author agreed to drop three derivable counters and to build the exited-plus-live stats infrastructure first, yet the CF entry still holds v1 (maintenance-fit-1, standards-compat-2). v1 also loses the counters of exited backends (maintenance-fit-2). Code review makes sense once a version exists on the agreed design.

Not checked: The code was not read, so the exact cost of the counter calls and the permission checks of pg_stat_get_backend_aio() are unchecked. Messages #4 to #8 were seen only through the brief and the objections.

## Prior attempts

- [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://www.postgresql.org/message-id/flat/CA+fd4k5_pPAYRTDrO2PbtTOe0eHQpBvuqmCr8ic39uTNmR49Eg@mail.gmail.com) (related; Reverted, 2020-07-13). "The stats with this commit was available only for WALSenders, however, users might want to see for backends doing logical decoding via SQL API." Different now: P5 tracked logical decoding spill stats; this tracks AIO counters. Both cover only some processes and lose data: v1 drops counters at backend exit and skips IO workers and several auxiliary processes.
- [Show WAL stats on pg_stat_io](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com) (related; Returned with feedback, 2024-01-18). "IMHO, perhaps we'd better put this patch aside until we are absolutely *sure* of what we want to achieve when it comes to WAL, and I am afraid that this cannot happen until we're happy with the way we handle WAL reads *and* writes, including WAL receiver or anything that has the idea of pulling its own page callback with XLogReaderAllocate() in the backend." ([message](https://www.postgresql.org/message-id/ZaTQSOVpwekhOXx-%40paquier.xyz)) Different now: P6 added WAL I/O to pg_stat_io and was set aside until the wider I/O stats design settled. Here the author agreed in message #7 to do the exited-plus-live stats redesign before the AIO counters.

## Objections a reviewer is likely to raise

### 1. v1 grows pgstat_backend.c, a direction the thread already questioned

A committer applies v1 from the CommitFest entry, and the AIO counters become one more stats family inside the backend stats kind. Each later per-backend counter set lands in the same file with its own flush flag. The planned exited-plus-live redesign then has to move or rewrite all of them.

In message #5, Andres Freund doubts that pgstat_backend.c is the right direction because it will collect "all kinds of stats". He proposes per-kind shared counters for exited backends plus live per-backend counters. In message #7 the author agreed to do that redesign first in a new thread. The CF entry still holds v1, which uses the design both sides set aside.

**Question:** Should this CF entry be withdrawn until the exited-plus-live infrastructure exists, so that the AIO counters are built on it rather than on pgstat_backend.c?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg), [#7](https://www.postgresql.org/message-id/alB7acgJ%2FWW7B1yC%40bdtpg); `maintenance-fit-1`</sub>

### 2. v1 column set conflicts with the agreed redesign

v1 is committed from the open CF entry. Monitoring tools then query started, executed_async and completed_self. The redesign in message #7 drops these derivable columns and adds a total across exited backends, so those queries break or change meaning in the next release.

The author agreed in message #6 to remove started, executed_async and completed_self, and agreed in message #7 to land the redesign first. The CF entry still offers v1 as "Needs review". In [P6] a pg_stat_io extension was set aside so that the view's meaning would not change between releases, and the same reason applies to this function's contract. Exposing v1 now would set a column list that the thread has already rejected.

Precedent: [Show WAL stats on pg_stat_io](https://www.postgresql.org/message-id/flat/CAN55FZ3AiQ+ZMxUuXnBpd0Rrh1YhwJ5FudkHg=JU0P+-W8T4Vg@mail.gmail.com) (Returned with feedback, 2024-01-18)

**Question:** Should this CF entry be withdrawn until a version exists with the reduced column set and an aggregate over exited backends?

<sub>fixable, level 2, design; messages [#5](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg), [#7](https://www.postgresql.org/message-id/alB7acgJ%2FWW7B1yC%40bdtpg); `standards-compat-2`</sub>

### 3. Completion columns change meaning with io_method, silently zero by default

A DBA on a server with io_method = worker, the default, reads completed_other = 0 and a small completed_self. The DBA concludes that completions are not shared or that IOs did not complete. The numbers never say that worker mode hides completions, so the DBA learns this only from a note in the docs.

The doc hunk calls completed_other meaningful only under io_uring, because IO workers do not track these statistics. Message #8 likely ran in worker mode, and it shows completed_self = 224 against executed_async = 26495. So completed_self also undercounts there, and the docs do not say so. Under the default setting the columns return plausible numbers with no sign that they are incomplete, and the meaning of the columns depends on a GUC.

**Question:** Under io_method = worker, what should completed_self and completed_other report, and why not return NULL or count completions on the worker side?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/akzcq9jtWK5C92g1%40bdtpg), [#8](https://www.postgresql.org/message-id/CAF0whufYfeLqsUy%2BUOoiYZgLpsPEYfXJm%3DqNocL-k69-yQbczw%40mail.gmail.com); `standards-compat-1`</sub>

### 4. Per-IO counter calls on AIO hot paths have no published measurement

Every server pays four function calls and counter updates per IO in pgaio_io_stage(), pgaio_io_process_completion() and pgaio_submit_staged(), whether or not anyone reads the function. On a high-IOPS io_uring workload, a small regression would appear only after release, with no numbers to compare against.

Message #3 asks for a review of these call sites because they run for all workloads by default. Message #1 reports no measurable regression but gives no workload or numbers. Andres Freund in message #5 says the overhead "might be ok" but questions the number of counters per IO. If ten monitoring patches each add a few calls per IO, the AIO path slows for everyone.

**Question:** What does a high-IOPS benchmark of small random reads under io_uring and worker mode show, with and without the reduced counter set?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/akzcq9jtWK5C92g1%40bdtpg), [#3](https://www.postgresql.org/message-id/ak3zpA2HHh5CxQK8%40paquier.xyz), [#5](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi); `maintenance-fit-3`</sub>

### 5. Per-backend-only function fixes a user surface the redesign will change

A DBA monitors pg_stat_get_backend_aio() across parallel queries. The parallel workers exit, so their counters disappear, and the totals undercount AIO activity. If a global view arrives in a later release, the existing function and its columns must stay, or users' monitoring queries break.

The proposal has no global view, loses the stats of exited backends, and reports nothing for the checkpointer, bgwriter, startup process and autovacuum launcher. Andres Freund says in message #5 that stats for exited backends are almost always wanted. [P5] was reverted partly because its stats covered only some process types and not all users who needed them. Shipping the narrow function first commits the project to supporting it next to whatever the redesign produces.

Precedent: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://www.postgresql.org/message-id/flat/CA+fd4k5_pPAYRTDrO2PbtTOe0eHQpBvuqmCr8ic39uTNmR49Eg@mail.gmail.com) (Reverted, 2020-07-13)

**Question:** Can the function and its column set be defined so that they stay unchanged once exited-backend totals and a global view exist?

<sub>fixable, level 1b, design; messages [#5](https://www.postgresql.org/message-id/et272fdhdx6yphlgzvrgsf7bgwnf3vqciwp4gxqubro42uaflp%40ohslaocvwgvi), [#6](https://www.postgresql.org/message-id/ak8hTtwna%2Buha7aq%40bdtpg); `maintenance-fit-2`</sub>

## Questions for the author

1. Should this CF entry be withdrawn or marked Waiting on Author until the exited-plus-live stats infrastructure exists and a version is built on it?
2. Which columns remain after dropping started, executed_async and completed_self, and will that list stay unchanged once exited-backend totals and a global view exist?
3. Under io_method = worker, what should completed_self and completed_other report, and why not return NULL or count completions in the IO workers?
4. What does a high-IOPS benchmark of small random reads show under io_uring and worker mode, with and without the reduced counter set?
5. How will users see AIO counters for parallel workers and other backends that have exited before the function is called?

## Touch points

- **catalog changes:** The patch adds a pg_proc.dat entry for pg_stat_get_backend_aio() and needs a catversion bump.
- **ABI / extension API:** pgstat_flush_backend() gets a new flag value, and pgstat.h gains the PgStat_AioCounters struct.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: v1 column set conflicts with the agreed redesign
- **new GUC:** see objection standards-compat-1: Completion columns change meaning with io_method, silently zero by default
- **maintenance burden ("who maintains this in 10 years"):** A new stats family in pgstat_backend.c and a seven-column function contract must be kept through the planned stats redesign.
- **performance on the hot path:** Counter calls run per IO in pgaio_io_stage(), pgaio_io_process_completion() and pgaio_submit_staged() for every workload.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund (AIO) and Michael Paquier (cumulative stats) take part in the thread; the owner needs to know both the AIO paths and the pgstat infrastructure.
- Priority: needs design rethink; audience niche; review cost medium. The counters serve people who tune AIO, and the 19.5 KB patch touches aio.c hot paths and the backend stats kind. The thread agreed to redesign the stats infrastructure first, so reviewing v1 now would spend time on a design both sides set aside.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
