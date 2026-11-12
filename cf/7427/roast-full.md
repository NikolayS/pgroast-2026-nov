# Design pre-review: [RFC PATCH v1] On-demand WAL replay: accept connections before crash recovery has applied the WAL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7427](https://commitfest.postgresql.org/patch/7427/) (Needs review); authors: Srinath Reddy (srinath-reddy-sadipiralla)
- Thread: [\[RFC PATCH v1\] On-demand WAL replay: accept connections before crash recovery has applied the WAL](https://www.postgresql.org/message-id/flat/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com), started 2026-10-10
- Messages read: 2 of 2
- Patch prose read: v1-0001-Replay-WAL-on-demand-after-a-crash-instead-of-bef.patch, v1-0002-Add-a-TAP-test-and-documentation-for-fast-crash-r.patch, v1-0001-Replay-WAL-on-demand-after-a-crash-instead-of-bef.patch, v1-0002-Add-a-TAP-test-and-documentation-for-fast-crash-r.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is stated and measured, but the design leaves the pending window undefined in several places. No checkpoint can complete during the drain and nothing bounds pg_wal growth (replication-ondisk-1). The proposal does not cover a redo PANIC in a backend (replication-ondisk-2), and nothing checks the new rules for redo routines (maintenance-fit-1). The simpler unwaited-checkpoint path is not compared (already-exists-1). All are fixable, and the thread has only the author's message.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that no path makes a page visible before it is current, or that failed replay cleans up correctly. The cost in StartReadBuffersImpl() with the setting off is also unmeasured.

## The proposal, restated

**Problem.** After a crash the server refuses connections until it has replayed all WAL since the last checkpoint and finished the end-of-recovery checkpoint. Both costs grow with WAL volume, so downtime after a crash grows with it. The target is any operator who wants to cut that downtime.

**Proposed design.** With fast_crash_recovery on, the startup process applies only records without block references and stores, per page, the LSNs of page-modifying records in a dshash table in DSM. The server then opens, and a page is replayed inside StartReadBuffersImpl() under BM_IO_IN_PROGRESS on its first read from disk. A new auxiliary process replays the remaining pages, then drops the index and requests a checkpoint.

**User-visible surface.**
- New GUC fast_crash_recovery (boolean, default off, server start only).
- New auxiliary process type "fast recovery worker" (log/backend type fastrecoveryworker).
- New wait event FAST_RECOVERY_DRAIN and new LWLocks/tranches LSNIndex, LSNIndexDSA, LSNIndexHash.
- Until the worker finishes, CHECKPOINT, pg_backup_start() and CREATE/DROP DATABASE fail with "checkpoint request failed", and clean shutdown waits.
- Expected output of the stats regression test changes, likely new pg_stat_io rows for the new process type.

**What the author claims.**
- Time to accept connections drops 1.9x to 5.5x in pgbench crash tests on a 6-vCPU VM, with the best case at full_page_writes = off.
- The first full scan of a hot table after opening is about 1.5 to 7 times slower, and the background drain costs more total time than stock recovery.
- Checks with per-table hashes, the tpcb balance invariant and pg_amcheck --heapallindexed found no differences against stock recovery.
- check-world passes with the setting forced on, except 018_wal_optimize, which can fail because it issues CHECKPOINT right after a crash restart.
- The design adds nothing new to the buffer manager protocol, and an exclusive lock can replace a cleanup lock because no backend can hold the page yet.

**Thread state.** Only the author has posted; message #2 is a forward of message #1. No reviewer has replied, and CFbot reports that the patch needs a rebase.

## 2. Prior attempts

### Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file (related)

- **Link:** [https://commitfest.postgresql.org/patch/3442/](https://commitfest.postgresql.org/patch/3442/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com)
- **Date:** decided 2022-02-27; started 2021-12-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier argued that the current state is not incorrect, because an end-of-recovery checkpoint is a shutdown checkpoint and the code has relied on that for about 10 years. He said the change was not checked against other assumptions, such as unlogged relation reset needing a clean shutdown, and that it leaves an invalid minimum recovery point in DB_IN_ARCHIVE_RECOVERY. He added that it would make back-patching harder in a sensitive area, and the entry was closed after the thread went quiet.
- **Reason (quote):** "This patch would bring some extra visibility for something that's not incorrect either on HEAD, as end-of-recovery checkpoints are the same things as shutdown checkpoints." ([message](https://www.postgresql.org/message-id/YhmaUxDMemv5hmba%40paquier.xyz))
- **What is different this time:** P14 changed the control file state around the end-of-recovery checkpoint. This proposal opens the server before any such checkpoint, so code that treats it as a shutdown checkpoint is affected.


## 3. Objections a reviewer is likely to raise

### 1. No checkpoint during the drain lets pg_wal and the redo window grow

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

A busy primary reopens after a crash and takes full write load while the worker drains for minutes. With CreateCheckPoint() refused, max_wal_size is not honoured and pg_wal can fill the disk, causing a PANIC. Any second crash replays from the old redo point, through all old and new WAL.

Message #1 refuses checkpoints until the LSN index is gone, and case C shows a 507 s drain on an idle server. Under load, the worker competes with clients, so the window has no stated upper bound. The proposal has no limit on WAL kept, no cap on drain time, and no fallback that forces a checkpoint. [P9] was reverted because committed changes stayed un-checkpointed with full reliance on the WAL stream, and this design creates the same state after every crash.

**Precedents.**
- [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (Reverted, 2012-06-13)

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-10-10)](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com)

**Question:** What bounds pg_wal growth and the redo window while the worker drains under write load, and what happens when that bound is reached?

Counter-check: held. CreateCheckPoint() refuses while the index exists, and the proposal gives no bound on pg_wal growth or on the drain under write load. [P9] was about an idle server with no end to the gap, so it must not be used as support.

### 2. Redo errors in backends can turn into a crash-restart loop

fixable · level 2, rubric item 7 · design, WAL / replication / standby · `replication-ondisk-2`

One page has a record whose redo hits a PANIC-level consistency check. In stock recovery, startup stops once with a clear message. Here the server opens, the first reader of that page triggers a cluster-wide crash restart, recovery reopens, and the next reader of the page crashes it again, each time over more WAL.

Many redo routines report inconsistent pages with PANIC, and the commit message covers only ERROR and FATAL cleanup in a backend. Moving redo into ordinary backends and the worker means a redo PANIC now resets all of shared memory, including the DSM index, while clients are connected. The proposal does not say how a page that fails replay is reported, isolated, or kept from restarting the cluster again and again.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-10-10)](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com)

**Question:** What happens when on-demand redo of one page raises PANIC, and how does the design stop repeated crash restarts over a growing WAL window?

Counter-check: held. Core redo routines do raise PANIC on inconsistent pages, and a PANIC in a backend triggers a crash restart. The commit message covers only ERROR and FATAL cleanup, so nothing stops the next reader of that page from repeating the crash.

### 3. Every redo routine, including custom rmgrs, gains unchecked new rules

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, WAL / replication / standby · `maintenance-fit-1`

A later patch, or an extension using a custom WAL resource manager, adds a redo routine that reads a second page or assumes all blocks of a record are applied together. With fast_crash_recovery on, that page is replayed wrongly or nests replay. The damage shows up after some production crash, as silent page corruption.

Message #1 makes correctness depend on new rules: redo of an indexed page never reads another indexed page, other blocks report BLK_DONE, initialized blocks go to a scratch local buffer, and an exclusive lock stands in for a cleanup lock. These hold for today's core redo routines by inspection, but nothing in the design checks them. The default is off and only crash tests exercise the feature, so a routine that breaks a rule passes check-world. Custom rmgrs are outside the project's control.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-10-10)](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com)

**Question:** What check, assertion or test fails when a new or extension redo routine breaks the one-page, no-nesting rule this design depends on?

Counter-check: held. The BLK_DONE handling is already required of redo routines, but the rule that redo of an indexed page reads no other page is new and unchecked. An assertion, or eager replay for custom rmgrs, could fix it in a later version, so it is fixable.

### 4. The existing promotion path already skips the waited end-of-recovery checkpoint

fixable · level 2, rubric item 12 · design, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator enables fast_crash_recovery and gets a deferred-replay buffer manager path and a new auxiliary process. Most of their downtime was the end-of-recovery checkpoint. Writing an end-of-recovery record and requesting the checkpoint without waiting, as promotion already does, would have removed that part with far less new state.

In case A the checkpoint is 9.2 of 15.1 s, and in B it is 30.4 of 96.1 s. Postgres already ends archive recovery at promotion with an end-of-recovery record and an unwaited checkpoint. The proposal does not compare against reusing that path for crash recovery. Only the gain beyond that simpler change justifies replay inside StartReadBuffersImpl() and the failing CHECKPOINT window.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-10-10)](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com)

**Question:** How much time to accept connections remains if crash recovery only ends with an end-of-recovery record and an unwaited checkpoint, and does that remainder justify on-demand page replay?

Counter-check: held. The tables already split redo from checkpoint time, and stock redo alone is 5.8 s against 2.7 s in A and 173 s against 106 s in C. The proposal does not discuss the simpler unwaited-checkpoint path, so the design comparison is still open.

### 5. CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on

fixable · level 2, rubric item 12 · backpatchability, new GUC, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

After a crash, a backup tool calls pg_backup_start() or pg_basebackup, or a deployment script runs CREATE DATABASE. Each gets "checkpoint request failed" for as long as the drain runs, which was over 8 minutes in run C. The operator finds out when the nightly backup or a provisioning job fails, with an error that does not name the cause.

With the GUC on, commands that work on every server today fail for an unbounded time after a crash. The GUC changes the semantics of user commands, beyond recovery timing. Message #1, item 4, leaves the choice between failing and waiting open and ships the error in v1. The project's own 018_wal_optimize test already trips on this, which suggests external scripts will too. A committer would likely ask for one defined contract and a specific error message before the GUC is exposed.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-10-10)](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com)

**Question:** Will CHECKPOINT, pg_backup_start() and CREATE/DROP DATABASE wait during the drain, and if they fail, what errcode and hint tell the operator why and for how long?

Counter-check: held. Message #1, item 4, raises this itself and leaves the choice open, so v1 has no settled contract. The error text "checkpoint request failed" does not name the cause, and 018_wal_optimize already fails on it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: No checkpoint during the drain lets pg_wal and the redo window grow |
| WAL / replication / standby | **yes** | Page redo moves out of the startup process, and CreateCheckPoint() refuses until the fast recovery worker finishes. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Redo routines, including those of custom WAL resource managers, run under new one-page rules; bufmgr.h changes. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on |
| new GUC | **yes** | Adds fast_crash_recovery, a boolean set at server start, default off. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on |
| maintenance burden ("who maintains this in 10 years") | **yes** | New auxiliary process, DSM index, and rules that every future redo routine must keep, across bufmgr, checkpointer and recovery. |
| performance on the hot path | **yes** | Adds a check in StartReadBuffersImpl(), the common read path; no measurement with the setting off. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What bounds pg_wal growth and the redo window while the worker drains under write load, and what does a second crash in that window replay?
2. What happens when on-demand redo of one page raises PANIC in a backend, and how does the design prevent repeated crash restarts on that page?
3. What assertion or test fails when a new redo routine, or one in a custom resource manager, reads a second page during on-demand redo?
4. How much time to accept connections remains if crash recovery only writes an end-of-recovery record and requests an unwaited checkpoint, as promotion does?
5. Will CHECKPOINT, pg_backup_start() and CREATE/DROP DATABASE wait or fail during the drain, and what do smart, fast and immediate shutdown each do?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows both the buffer manager read path and WAL recovery with checkpoints; none has replied in the thread.
- **Liability:** The owner signs for the correctness of page redo inside backends after every crash, for the new redo-routine rules, and for the checkpoint-free window, for as long as the setting exists.
- **Author follow-up:** The author plans more measurements on EC2 and has a waiting variant written. CFbot has asked for a rebase since 2026-10-18 and no new version is posted.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** A single 94 KiB patch touches bufmgr, checkpointer, postmaster and recovery, and the behaviour of the pending window is still open. The thread cites no field reports, and the feature excludes standby and archive recovery, so the size of the audience is not shown.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Clean shutdown can block for minutes, beyond service manager stop timeouts. Not listed: not among the strongest objections.
- `standards-compat-3` (fixable, level 2) Pending-recovery window is not visible to clients or tools. Not listed: not among the strongest objections.
- `maintenance-fit-4` (fixable, level 2) One all-or-nothing patch in bufmgr and recovery with no reviewer. Not listed: not among the strongest objections.
- `demand-1` (fixable, level 1b) No field evidence that crash-restart downtime is a pressing user problem. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` No checkpoint can complete during the drain window. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same checkpoint gap with a more concrete failure. [P9] concerned an idle server skipping checkpoints with no end, so it is weak support here.
- `prior-attempts-2` Clean shutdown after a crash restart waits for the full drain. Refuted (not specific): Duplicate of standards-compat-2. [P10] was withdrawn because it did not fix slow regular checkpoints, which is a different reason from a shutdown that waits for the drain.
- `replication-ondisk-3` Switchover and replica rebuild are blocked until the drain ends. Refuted (not specific): Duplicate of standards-compat-2 for the shutdown wait and of standards-compat-1 for the pg_backup_start() failure. The HA framing adds no separate failure beyond those two.
- `maintenance-fit-2` No checkpoint during the drain lets WAL and second-crash recovery grow. Refuted (not specific): Duplicate of replication-ondisk-1, which covers pg_wal growth and the second-crash redo window with the same facts.
- `maintenance-fit-3` New failure modes reach backup, CREATE DATABASE and tooling. Refuted (not specific): Duplicate of standards-compat-1 for the failing commands and of standards-compat-3 for the missing progress view.
- `already-exists-2` Checkpoint spacing already bounds crash recovery WAL; benchmarks disable it. Refuted (no failure scenario): Cases A, B and D replay 1.8 to 2.7 GiB, which is within the "few GiB" the objection grants a tuned server. A crash window of that size is the same input whether or not earlier checkpoints ran; the tuning trade-off is covered by demand-1.
- `demand-2` Faster connect time is paid for by a long degraded window. Refuted (already handled): Message #1 states the trade of more total work for earlier availability, and the setting is off by default, so the slow window falls only on sites that enable it. The failing commands and the redo rule are covered by standards-compat-1 and maintenance-fit-1.

### Review angles

- prior-attempts: 2 objection(s). No precedent proposes on-demand page redo after crash recovery. The two objections rest on reasons from related attempts that still apply.
- standards-compat: 3 objection(s). Not applicable: three objections from the standards and compatibility angle.
- replication-ondisk: 3 objection(s). No on-disk format, catalog or wire protocol change is proposed. The objections concern WAL retention, crash restarts and replication tooling during the pending window.
- security: 0 objection(s). The proposal grants no new privilege and exposes no data, query text or file access to non-superusers; the GUC is server-start only and off by default, and replay sits below the RLS and permission layer. Not checkable without code: whether an unprivileged session can delay a hot page's replay for other readers by cancelling it repeatedly with statement_timeout.
- maintenance-fit: 4 objection(s). Objections were found; see the list.
- already-exists: 2 objection(s). No extension or external tool does on-demand per-page redo. Both objections concern existing ways to reduce part of the downtime.
- demand: 2 objection(s). Two demand objections raised; both are "show demand" in kind, not proof that the idea lacks users.

### Precedents retrieved

- P1: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P2: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P3: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P4: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant; reason summarised from its thread
- P5: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant; reason summarised from its thread
- P6: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant; reason summarised from its thread
- P7: [Add function - pg_accept_connections_start_time()](https://commitfest.postgresql.org/patch/5576/) (cf, Returned with feedback, 2025-07-25) - judged not relevant; reason summarised from its thread
- P8: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P9: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - used in the review
- P10: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - used in the review; reason summarised from its thread
- P11: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - used in the review
- P12: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - used in the review; reason summarised from its thread
- P13: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - judged not relevant; reason summarised from its thread
- P14: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - used in the review; reason summarised from its thread
- P15: [Log LSN before table is dropped](https://commitfest.postgresql.org/patch/5386/) (cf, Returned with feedback, 2025-07-25) - judged not relevant
- P16: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant
- P17: [Key management with tests](https://commitfest.postgresql.org/patch/2925/) (cf, Withdrawn, 2021-07-15) - judged not relevant
- P18: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P19: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (cf, Withdrawn, 2024-01-31) - judged not relevant
- P21: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant
- P22: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P23: [VACUUM memory management](https://commitfest.postgresql.org/patch/2374/) (cf, Returned with feedback, 2020-07-02) - judged not relevant
- P24: [Revert "Use WAIT FOR LSN in PostgreSQL::Test::Cluster::wait_for_catchup()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e54ce0b2da62ac71e3b2a5212f32e82b07bf39a5) (revert, Reverted, 2026-01-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 217 s; cost reported by the CLI: $3.09
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 134 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:54:43.658Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
