# Design pre-review: Redesign per-backend statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7091](https://commitfest.postgresql.org/patch/7091/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Redesign per-backend statistics](https://www.postgresql.org/message-id/flat/anCjoFkGah%2FuB7lw%40bdtpg), started 2026-08-03
- Messages read: 15 of 15
- Patch prose read: v5-0001-pgstat-add-tests-for-per-backend-statistics.patch, v5-0002-pgstat-add-new-infrastructure-for-per-backend-sta.patch, v5-0003-pgstat-move-WAL-statistics-to-new-per-backend-inf.patch, v5-0004-pgstat-move-lock-statistics-to-new-per-backend-in.patch, v5-0005-pgstat-move-I-O-statistics-to-new-per-backend-inf.patch, v5-0006-pgstat-remove-PGSTAT_KIND_BACKEND.patch, v1-0001-pgstat-add-tests-for-per-backend-statistics.patch, v1-0002-pgstat-add-new-infrastructure-for-per-backend-sta.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b, and two committers who know pgstat asked for the fixed array. Two fixable level 2 points remain. maintenance-fit-1 asks for query-latency numbers for the array version, because the thread only measured the dshash version. standards-compat-1 asks what a shared reset does to each live entry's stats_reset.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the locking on unused slots and the transfer at exit, ProcNumber reuse and shutdown could not be checked for lost or double-counted counters. Only 3 of 15 messages were shown, so some open questions may already have answers in the thread.

## The proposal, restated

**Problem.** Per-backend WAL, lock and I/O statistics are reported and stored twice, once in the global fixed statistics and once in PGSTAT_KIND_BACKEND. Reviewers raised this duplicated work, and its cost grows as more statistics gain per-backend reporting. The current design also excludes auxiliary processes such as the checkpointer, background writer, startup process and autovacuum launcher.

**Proposed design.** Each participating fixed-numbered kind keeps a shared accumulator for exited processes plus a ProcNumber-indexed shared array, sized at startup, for live processes, each entry with its own LWLock. Flushes update only the live entry, global views add all live entries to the accumulator, and entries move into the accumulator at exit, ProcNumber reuse and clean shutdown. WAL, lock and I/O move over, and PGSTAT_KIND_BACKEND is removed.

**User-visible surface.**
- pg_stat_get_backend_wal(), pg_stat_get_backend_lock() and pg_stat_get_backend_io() now return rows for the checkpointer, background writer, startup process, autovacuum launcher and shared-memory-only workers.
- pg_stat_reset_shared('wal'), ('lock') and ('io') now also clear the per-backend statistics of every live process.
- Queries on pg_stat_wal, pg_stat_lock and pg_stat_io now add up all live per-backend entries, so they cost more with many backends.
- PgStat_KindInfo gains per-backend metadata, and registering a custom statistics kind with per-backend entries fails with an error.
- PGSTAT_KIND_BACKEND is removed, and pgstat_backend.c now holds the generic per-backend infrastructure.

**What the author claims.**
- Each counter is reported and stored once.
- Flushes no longer take the global statistics lock or do a hash lookup, which removes global lock contention from the flush path.
- WAL, lock and I/O get independent storage and locking, so adding a new per-backend statistic adds no contention to existing kinds.
- With 5000 maximum backends the fixed array takes about 16 MB; the earlier dshash version used 22 MB with 3735 sessions.
- In the dshash version, global view queries went from about 0.02-0.07 ms to 0.35-2.8 ms at 10000 backends; the author calls this acceptable.

**Thread state.** Michael Paquier and Andres Freund pushed for a fixed array over dshash, and v4 adopted it; v5 addresses Michael's review of v4. Open items are whether to keep the snapshot TAP test and when to support custom per-backend statistics, which the author leaves for a follow-up.

## 2. Prior attempts

### Get rid of pgstat_count_backend_io_op*() functions (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6025/](https://commitfest.postgresql.org/patch/6025/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aLWonxez0MBSqEvA@ip-10-97-1-34.eu-west-3.compute.internal](https://www.postgresql.org/message-id/flat/aLWonxez0MBSqEvA@ip-10-97-1-34.eu-west-3.compute.internal)
- **Date:** decided 2026-03-25; started 2025-09-01
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier, Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier objected to the ordering dependency between stats kinds at flush time, which so far have been independent, and said the copy approach hurts readability. He also said there was no benchmark data to choose among inlining, static structures, copies or fewer diffs, so the picture was incomplete. Andres Freund rejected the alternative WalUsage-style diff approach as too costly. The thread ends with the author disagreeing, and the visible messages do not show a final decision.
- **Reason (quote):** "Hmm. I really have an issue with the ordering dependency this introduces between stats kinds, assumed to happen inside the code." ([message](https://www.postgresql.org/message-id/aNTkWRZAg9FCBKy0%40paquier.xyz))
- **What is different this time:** Both remove the double counting of per-backend I/O, but P1 copied pending I/O stats into the backend kind and needed a flush order between kinds. Here each kind owns its own ProcNumber-indexed entries, so no copy and no flush ordering remain.


## 3. Objections a reviewer is likely to raise

### 1. Global view query cost scales with MaxBackends, unmeasured for the array design

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A site with max_connections = 5000 runs a monitoring agent that polls pg_stat_io every few seconds on each node. Every query now takes one shared LWLock per array slot and adds up every entry. The operator sees higher monitoring latency, and more time spent in LWLock calls, only after upgrading.

The only latency numbers are for the dshash version (message #5), where pg_stat_io went from about 0.07 ms to 2.8 ms at 10000 backends. Message #10 says a fixed array must also visit unused slots, and the v4/v5 array design has no numbers at all. pg_stat_io, pg_stat_wal and pg_stat_lock are read by nearly every monitoring tool, so the cost falls on everyone, including people who never call pg_stat_get_backend_*(). Each later per-backend kind adds another scan of this kind.

Thread: [#4 (Michael Paquier, 2026-08-10)](https://www.postgresql.org/message-id/anlVx_TD_Q8He5uf%40paquier.xyz), [#5 (Bertrand Drouvot, 2026-08-10)](https://www.postgresql.org/message-id/annj%2FETYOFgbrGsn%40bdtpg), [#10 (Bertrand Drouvot, 2026-09-22)](https://www.postgresql.org/message-id/arHw6cycX%2Bpe53wX%40bdtpg), [#12 (Bertrand Drouvot, 2026-10-26)](https://www.postgresql.org/message-id/at8%2FezwmjHCWRmh2%40bdtpg), [#13 (Michael Paquier, 2026-11-05)](https://www.postgresql.org/message-id/auwR7e5mba17AISr%40paquier.xyz)

**Question:** What do pg_stat_io, pg_stat_wal and pg_stat_lock cost with v5 at a few common max_connections values, with most slots idle or unused, compared with master?

Counter-check: held. The thread's only latency numbers are for the dshash version, and the array version scans every slot. The claim of one LWLock per unused slot is not established, and two committers who know the area asked for the array, so this is a request for numbers.

### 2. Shared reset now clears live per-backend counters without a stated contract

fixable · level 2, rubric item 10 · design, backpatchability · `standards-compat-1`

A monitoring agent samples pg_stat_get_backend_io() for each pid and graphs deltas. A DBA runs pg_stat_reset_shared('io'), and every live backend's counters drop to zero. Unless the per-backend stats_reset timestamp also changes, the agent reports negative or wrapped deltas and finds out only from wrong graphs after upgrade.

On master, a shared reset leaves per-backend counters alone, and pg_stat_reset_backend_stats() resets them. The doc hunks in 0003-0005 add the new reach of pg_stat_reset_shared() but do not say whether each live entry's stats_reset is updated. Message #1 calls this "a documented behavior change", but the thread has no release-note text and no rule for how clients tell a shared reset from a per-backend one. Tools that already use the per-backend functions upgrade into this silently.

Thread: [#1 (Bertrand Drouvot, 2026-08-03)](https://www.postgresql.org/message-id/anCjoFkGah%2FuB7lw%40bdtpg)

**Question:** After pg_stat_reset_shared('io'), does each live backend's per-backend stats_reset change, and what release-note text tells tool authors about the new reset reach?

Counter-check: held. The doc hunks in 0003-0005 state the new reset reach, but nothing says whether each live entry's stats_reset changes after a shared reset. The release-note request should be dropped, because Postgres patches do not carry release-note text, and backpatching does not apply.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0005 touches walsender.c, and per-backend rows now cover the startup process and WAL-related auxiliary processes. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | PgStat_KindInfo gains per-backend metadata, PGSTAT_KIND_BACKEND is removed, and custom kinds with per-backend entries fail to register. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Shared reset now clears live per-backend counters without a stated contract |
| maintenance burden ("who maintains this in 10 years") | **yes** | New generic per-backend infrastructure with per-entry LWLocks and transfer at exit, ProcNumber reuse and clean shutdown. |
| performance on the hot path | **yes** | Flushes change locking, and pg_stat_wal, pg_stat_lock and pg_stat_io now add up every array slot. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What do pg_stat_io, pg_stat_wal and pg_stat_lock cost with v5 at a few common max_connections values, with most slots unused, compared with master?
2. After pg_stat_reset_shared('io'), does each live backend's per-backend stats_reset change, and where do the docs say so?
3. Is there a flush-path benchmark against master that shows the reduced lock contention claimed in message #1?
4. How much shared memory do the three arrays use at common max_connections values, and does the size grow with max_worker_processes and other process slots?
5. Does a global view query take the LWLock of unused array slots, or does it skip them without locking?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier reviews the series in the visible thread, and the commit messages list him and Andres Freund as reviewers; both asked for the fixed array.
- **Liability:** The owner signs for the correctness of global totals across process exit, ProcNumber reuse and clean shutdown, for the query cost of three widely polled views, and for the changed reset behaviour, for as long as the cumulative statistics code lives.
- **Author follow-up:** The author posted five versions in about three months and changed the design from dshash to a fixed array after review, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The series removes duplicate accounting that reviewers raised and unblocks further per-backend statistics, and committers who know pgstat are engaged. Six staged patches stay within the pgstat code, and the two open points need numbers and one documented answer.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The one related attempt, P1, failed on a flush-order dependency between stats kinds, and this design has no such dependency. Message #1 cites the earlier discussion that raised the double counting, and message #14 says the current design blocks new per-backend stats, so the thread itself starts from that history.
- standards-compat: 1 objection(s). Other user-visible changes look compatible: per-backend rows for auxiliary processes only add data, and the custom-kind registration error blocks a feature that does not exist today.
- replication-ondisk: 0 objection(s). The proposal adds no on-disk format change, WAL record, replication state, catalog change or protocol message, and no persistent state outside the stats file. Live entries move into the global accumulators before a clean shutdown writes the stats file, and crash recovery discards stats as before. The PgStat_KindInfo change is an extension API change for a major release only, with no backpatch planned.
- security: 0 objection(s). No objection from the security angle. The proposal adds no new SQL function, privilege or GUC. The per-backend functions and global views keep their current access rules. The new rows for the checkpointer, background writer, startup process and autovacuum launcher hold counters that pg_stat_io already shows per backend type. pg_stat_reset_shared() keeps its existing privilege check; it now also clears live per-backend entries, and only callers who can already reset shared statistics can do that. The extra per-entry shared locking on global view queries is a cost question for the performance angle. Ordinary flushes use a conditional lock acquire, so an unprivileged reader does not block other backends' routine flushes. The material does not say who may call pg_stat_get_backend_*() for other users' backends, and it does not change that.
- maintenance-fit: 1 objection(s). Both committers who know pgstat, Michael Paquier and Andres Freund, are engaged and chose the fixed array, so ownership and the basic design look settled. The patch series is staged, with tests first. The memory cost was accepted in message #13, and support for custom per-backend kinds is deferred as a follow-up.
- already-exists: 0 objection(s). No existing feature, extension or tool meets this need. The proposal replaces PGSTAT_KIND_BACKEND, so it does not add a second mechanism, and the reviewers in messages #8, #9 and #13 settled the choice of a fixed array over keeping that kind. [P1] tried a different fix for the same double counting and does not show an existing alternative.
- demand: 0 objection(s). No demand objection. The problem is internal: duplicate accounting in the stats system that reviewers flagged. Andres Freund proposed this design and Michael Paquier backs the fixed array (messages #1, #8, #9, #13), so the request comes from the people who would own the area. Missing flush-path and fixed-array query numbers belong to the performance angle.

### Precedents retrieved

- P1: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - used in the review; reason summarised from its thread
- P2: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P3: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant; reason summarised from its thread
- P4: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P5: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P6: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant; reason summarised from its thread
- P7: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P8: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P9: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant; reason summarised from its thread
- P10: [Rather than mmap()-ing in 8k pages, mmap()'ing entire files into a backend.](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Miscellaneous_Performance) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P12: [Revert pg_bind_textdomain_codeset to a existant-but-empty function when ENABLE_NLS is not defined, for better compatibility of the backend with modules...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e43fd89762a5a894ea04f4fe74e05f95a1488664) (revert, Reverted, 2009-03-09) - judged not relevant
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [accounting for memory used for BufFile during hash joins](https://commitfest.postgresql.org/patch/2109/) (cf, Returned with feedback, 2019-11-26) - judged not relevant
- P15: [Memory Accounting (towards memory-bounded HashAgg)](https://commitfest.postgresql.org/patch/269/) (cf, Rejected, 2015-07-26) - judged not relevant
- P16: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant
- P17: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P20: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant
- P21: [Statistics for array types](https://commitfest.postgresql.org/patch/323/) (cf, Returned with feedback, 2016-03-18) - judged not relevant
- P22: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P23: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [bitmap cost should account for correlated indexes](https://commitfest.postgresql.org/patch/2310/) (cf, Returned with feedback, 2021-02-01) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0003-pgstat-move-WAL-statistics-to-new-per-backend-inf.patch not read
- attachment limit (8) reached; v1-0004-pgstat-move-Lock-statistics-to-new-per-backend-in.patch not read
- attachment limit (8) reached; v1-0005-pgstat-move-IO-statistics-to-new-per-backend-infr.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 96 s; cost reported by the CLI: $2.62
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:19:05.642Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
